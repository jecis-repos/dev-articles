---
title: "My Staging System Leaked 6 Orphaned Databases Every Monday — Saga Pattern Fixed It"
published: false
description: "A 16-step provisioning pipeline with compensating transactions. When step 14 fails, only steps 13 through 1 get rolled back. Zero orphaned resources."
tags: devops, typescript, infrastructure, patterns
series: "Build Your Own Staging System"
cover_image:
canonical_url:
---

I've been running an automated PR staging system in production for over three months now. Every pull request gets its own isolated environment — database, containers, reverse proxy config, the works. Zero manual intervention.

The first version leaked resources like a sieve. A DNS timeout at step 11 would leave behind a running container, a dangling database, and a Caddy config block pointing at nothing. I'd SSH in on Monday morning to find six orphaned Postgres databases and a compose file that looked like abstract art.

The fix wasn't retry logic. It was the Saga pattern.

## What's a Saga, and why should you care?

The Saga pattern comes from distributed systems literature — specifically Hector Garcia-Molina's 1987 paper. The core idea: when you have a long-running transaction that spans multiple services or resources, you can't just wrap it in a single atomic commit. Instead, each step gets a *compensating transaction* — a way to undo what it did.

If step 7 out of 16 fails, you don't nuke everything and start over. You run the compensating transactions for steps 6 through 1, in reverse order. Only the steps that actually completed get reversed.

This matters for infrastructure provisioning because:

1. **Resources are expensive to create.** Re-provisioning from scratch because step 14 had a transient network error is wasteful.
2. **Partial state is dangerous.** A database without a container is just wasted disk. A container without a reverse proxy entry is unreachable. A reverse proxy entry without a container returns 502s.
3. **Rollback can fail too.** If your rollback crashes halfway, you've made the mess worse.

## The provisioning pipeline

My staging system provisions a new environment in 16 steps. Here are the ones that matter for rollback:

1. Generate environment variables from the PR metadata
2. Create a `.env` file on disk
3. Add the service block to `docker-compose.yml`
4. Create the Postgres database
5. Add the Caddy reverse proxy block
6. Update the internal service registry
7. Update `/etc/hosts` (for internal resolution)
8. Pull the container image
9. Start the container
10. Run database migrations
11. Seed test data
12. Health check the running service

Each of these can fail. Each of them leaves behind something that needs cleanup.

## Tracking what's been committed

The key data structure is dead simple — a struct of boolean flags:

```typescript
interface ProvisionState {
  dockerComposeUpdated: boolean;
  databaseCreated: boolean;
  caddyBlockAdded: boolean;
  envGenerated: boolean;
  registryUpdated: boolean;
  hostsUpdated: boolean;
  containerStarted: boolean;
  healthChecked: boolean;
}
```

Every flag starts `false`. After each step *successfully completes*, its flag flips to `true`. If step 5 throws, `caddyBlockAdded` is still `false` — so rollback won't try to remove a Caddy block that was never added.

This sounds trivially obvious. It is. That's what makes it work. No complex state machines, no transition tables, no event sourcing. Just booleans.

```typescript
async function provisionEnvironment(slug: string, pr: PullRequest) {
  const state: ProvisionState = {
    dockerComposeUpdated: false,
    databaseCreated: false,
    caddyBlockAdded: false,
    envGenerated: false,
    registryUpdated: false,
    hostsUpdated: false,
    containerStarted: false,
    healthChecked: false,
  };

  try {
    await generateEnvFile(slug, pr);
    state.envGenerated = true;

    await addAppService(slug, pr.branch);
    state.dockerComposeUpdated = true;

    await createDatabase(slug);
    state.databaseCreated = true;

    await addCaddyBlock(slug);
    state.caddyBlockAdded = true;

    await updateRegistry(slug, pr);
    state.registryUpdated = true;

    await updateHosts(slug);
    state.hostsUpdated = true;

    await startContainer(slug);
    state.containerStarted = true;

    await healthCheck(slug);
    state.healthChecked = true;

    log.info(`Provisioned ${slug} successfully`);
  } catch (error) {
    log.error(`Provision failed at step for ${slug}`, error);
    await rollbackProvision(slug, state);
    throw error;
  }
}
```

## The rollback function

Here's where the pattern earns its keep:

```typescript
async function rollbackProvision(slug: string, state: ProvisionState) {
  log.info(`Rolling back provision for ${slug}`);

  if (state.containerStarted) {
    try {
      await stopContainer(slug);
      log.info(`rollback: stopped container for ${slug}`);
    } catch (e) {
      log.warn('rollback: stop container failed', e);
    }
  }

  if (state.hostsUpdated) {
    try {
      await removeHostsEntry(slug);
      log.info(`rollback: removed hosts entry for ${slug}`);
    } catch (e) {
      log.warn('rollback: hosts cleanup failed', e);
    }
  }

  if (state.registryUpdated) {
    try {
      await deregisterService(slug);
      log.info(`rollback: deregistered ${slug}`);
    } catch (e) {
      log.warn('rollback: registry cleanup failed', e);
    }
  }

  if (state.caddyBlockAdded) {
    try {
      await removeCaddyBlock(slug);
      log.info(`rollback: removed Caddy block for ${slug}`);
    } catch (e) {
      log.warn('rollback: caddy cleanup failed', e);
    }
  }

  if (state.databaseCreated) {
    try {
      await dropDatabase(slug);
      log.info(`rollback: dropped database for ${slug}`);
    } catch (e) {
      log.warn('rollback: database cleanup failed', e);
    }
  }

  if (state.dockerComposeUpdated) {
    try {
      await removeAppService(slug);
      log.info(`rollback: removed compose service for ${slug}`);
    } catch (e) {
      log.warn('rollback: compose cleanup failed', e);
    }
  }

  if (state.envGenerated) {
    try {
      await removeEnvFile(slug);
      log.info(`rollback: removed env file for ${slug}`);
    } catch (e) {
      log.warn('rollback: env cleanup failed', e);
    }
  }

  log.info(`Rollback complete for ${slug}`);
}
```

Three things to notice:

### 1. Reverse order

Steps are rolled back in the opposite order they were applied. This matters because later steps often depend on earlier ones. You stop the container before you remove the compose service that defines it. You drop the database before you remove the compose entry that might reference it.

### 2. Every rollback step has its own try/catch

This is non-negotiable. If removing the Caddy block fails (maybe the file is locked), you still want to drop the database and clean up the compose file. A rollback that aborts on first error is worse than no rollback at all — it gives you a false sense of safety while leaving behind a partially-cleaned mess.

The `log.warn` calls are important too. When you're debugging at 2am, you need to know that rollback *tried* to remove the Caddy block but couldn't. That's your breadcrumb trail.

### 3. Only completed steps get rolled back

If provisioning failed at `createDatabase`, then `caddyBlockAdded` is `false`. The rollback skips the Caddy removal entirely. No wasted work, no errors from trying to remove something that doesn't exist.

## Why not "delete everything and retry"?

I've seen systems that handle failure by nuking the entire environment and re-running provisioning from scratch. This is appealing in its simplicity, but it breaks in practice:

**It's slow.** Pulling a Docker image can take 30 seconds. Creating a database and running migrations takes another 20. If step 14 failed due to a transient DNS timeout, re-running all 16 steps wastes a minute of CI time per retry.

**It can cascade.** If your "delete everything" step fails halfway (network issues don't discriminate between provision and cleanup), you now have an *even more* confusing partial state.

**It doesn't compose.** When you have 15 PRs open and each is provisioning concurrently, "delete everything" might interfere with other environments that share resources (like the compose file or Caddy config).

**It masks bugs.** If step 7 consistently fails, retry-from-scratch makes it look like a flaky test. Saga rollback preserves the failure context — you see exactly which step failed and what the state was.

## The edge case that convinced me

Two weeks after deploying the saga-based rollback, I hit an edge case that would have been catastrophic without it.

A provisioning run had completed 14 of 16 steps when the health check timed out. The container was running but returning 503s because the database migration had a subtle bug. The rollback fired: stopped the container, removed the Caddy block, dropped the database, cleaned the compose file. All clean.

Under the old "nuke and retry" approach, the retry would have tried to create a database that already existed (because the nuke failed to drop it — the connection was timing out). That would have thrown a "database already exists" error, which the nuke handler would have tried to handle by... running the nuke again. I've seen this loop before. It doesn't end well.

## Making it production-solid

A few things I added after the initial implementation:

- **Timeout per rollback step.** If `dropDatabase` hangs for 30 seconds, something is wrong. Each compensating transaction gets its own timeout (configurable, defaults to 10 seconds).
- **Rollback telemetry.** I log which steps were rolled back and how long each took. This surfaces patterns — if Caddy block removal consistently takes 5 seconds, something's wrong with the file locking.
- **Idempotent compensating transactions.** `dropDatabase` checks if the database exists before trying to drop it. `removeCaddyBlock` checks if the block is present. This means running rollback twice is safe.

## Takeaway

The Saga pattern isn't exotic distributed systems theory. It's a practical tool for any multi-step process where partial completion is worse than no completion. If you're building infrastructure automation — staging systems, CI pipelines, deployment tooling — and you're not tracking what you've committed and rolling it back in reverse on failure, you're accumulating orphaned resources. Maybe slowly enough that you don't notice for months.

Until you do.

---

*Next up in Part 2: How to edit Docker Compose YAML files without a regex that works 95% of the time and destroys your formatting the other 5%.*
