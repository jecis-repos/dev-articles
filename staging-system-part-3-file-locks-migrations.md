---
title: "Two CI Runners, One Config File — The File Lock That Saved My Staging Server"
published: false
description: "POSIX advisory locks with stale PID recovery, SHA-256 migration divergence detection, and a gap-filling port allocator. The concurrency chapter."
tags: devops, typescript, linux, database
series: "Build Your Own Staging System"
cover_image:
canonical_url:
---

In Part 1, we covered saga-based rollback for provisioning pipelines. In Part 2, we built a safe YAML editing layer for Docker Compose files. But we glossed over a critical question: what happens when two pull requests get pushed at the same time?

The answer, if you're not careful, is file corruption.

My staging system runs on a single server. Multiple CI runners can trigger provisioning simultaneously. They all need to modify the same `docker-compose.yml`, the same Caddy config, and the same port registry. Without coordination, two concurrent writes produce a file containing half of each edit — and Docker Compose cheerfully refuses to parse the result.

This article covers three concurrency problems and their solutions: file locking, migration divergence detection, and gap-filling port allocation.

## POSIX advisory file locks with stale detection

Linux provides `flock(2)` for advisory file locking. "Advisory" means the kernel doesn't enforce it — any process can ignore the lock and write to the file anyway. But if every writer goes through your locking layer, advisory locks are sufficient and dramatically simpler than mandatory locking.

Here's the locking primitive:

```typescript
import { open, close } from 'fs';
import { flock } from 'fs-ext'; // npm install fs-ext

interface LockHandle {
  fd: number;
  lockPath: string;
  pid: number;
}

async function acquireLock(
  resource: string,
  timeoutMs: number = 30_000
): Promise<LockHandle> {
  const lockPath = `/var/run/staging/${resource}.lock`;
  const startTime = Date.now();

  while (true) {
    // Check for stale locks before attempting
    await cleanStaleLock(lockPath);

    try {
      const fd = await openFile(lockPath, 'w');
      await flockExclusive(fd);

      // Write our PID to the lock file for stale detection
      const pidBuffer = Buffer.from(String(process.pid));
      await writeFile(fd, pidBuffer);

      return { fd, lockPath, pid: process.pid };
    } catch (error) {
      if (Date.now() - startTime > timeoutMs) {
        throw new Error(
          `Timed out waiting for lock on ${resource} after ${timeoutMs}ms`
        );
      }
      // Wait briefly before retrying
      await sleep(100);
    }
  }
}

function releaseLock(handle: LockHandle) {
  try {
    close(handle.fd);
  } catch (e) {
    log.warn(`Failed to release lock ${handle.lockPath}`, e);
  }
}
```

### Stale lock detection

The most important part of any locking system is what happens when a lock holder dies. If the CI runner crashes mid-provision (OOM kill, network timeout, power loss), the lock file sits there forever. Every subsequent provisioning attempt times out waiting for a lock that will never be released.

The solution is PID-based stale detection:

```typescript
import fs from 'fs';

async function cleanStaleLock(lockPath: string): Promise<void> {
  if (!fs.existsSync(lockPath)) return;

  let content: string;
  try {
    content = fs.readFileSync(lockPath, 'utf8').trim();
  } catch {
    return; // Can't read lock file, skip
  }

  const pid = parseInt(content, 10);
  if (isNaN(pid)) {
    // Lock file exists but has no valid PID — treat as stale
    fs.unlinkSync(lockPath);
    log.warn(`Removed stale lock (no PID): ${lockPath}`);
    return;
  }

  // Check if the process is still alive
  if (!isProcessAlive(pid)) {
    fs.unlinkSync(lockPath);
    log.warn(`Removed stale lock (PID ${pid} dead): ${lockPath}`);
  }
}

function isProcessAlive(pid: number): boolean {
  try {
    // Signal 0 doesn't actually send a signal — it just checks
    // whether the process exists and we have permission to signal it
    process.kill(pid, 0);
    return true;
  } catch (error: any) {
    if (error.code === 'ESRCH') {
      // No such process
      return false;
    }
    if (error.code === 'EPERM') {
      // Process exists but we don't have permission to signal it
      // This means it's alive — owned by another user
      return true;
    }
    return false;
  }
}
```

The `process.kill(pid, 0)` trick is a POSIX standard: signal 0 doesn't deliver any signal, but the kernel still checks whether the target process exists. If it does, `kill` returns successfully. If it doesn't, you get `ESRCH`. If it exists but belongs to another user, you get `EPERM` — which means it's alive.

### Using locks in practice

Every file-modifying operation wraps itself in the lock:

```typescript
async function withLock<T>(
  resource: string,
  fn: () => Promise<T>
): Promise<T> {
  const handle = await acquireLock(resource);
  try {
    return await fn();
  } finally {
    releaseLock(handle);
  }
}

// Usage
async function addServiceSafely(slug: string, config: ServiceConfig) {
  return withLock('docker-compose', async () => {
    // Read, modify, validate, write — all within the lock
    const doc = loadComposeDocument();
    addServiceToDocument(doc, slug, config);
    await writeAndValidateCompose(doc);
  });
}
```

The `withLock` wrapper guarantees that even if the inner function throws, the lock is released. This composes with the saga rollback from Part 1: if provisioning fails at step 7, rollback acquires the compose lock, removes the service, and releases it.

### Why not a database-backed lock?

You might wonder why I'm using file locks instead of Redis `SETNX`, Postgres advisory locks, or etcd leases. The answer is dependency minimization. My staging system already depends on the filesystem being available (it's reading and writing compose files). Adding a network-reachable lock service introduces a new failure mode — what happens when Redis is down but the filesystem is fine? Now you can't provision because your lock service is unavailable, even though the resource you're protecting is perfectly accessible.

File locks work on a single server. My staging system runs on a single server. When the architecture outgrows this (multiple staging servers), I'll switch to a distributed lock. But not before I need to.

## Out-of-order migration detection

Here's a scenario that took me weeks to understand:

1. Developer A creates PR #42, branching from `main` at commit `abc123`
2. Developer A adds migration `005_add_user_roles.sql`
3. Developer B creates PR #43, also branching from `main` at `abc123`
4. Developer B adds migration `005_add_audit_log.sql`
5. Both PRs get staging environments
6. PR #43 gets merged first
7. PR #42's staging environment now has a migration history that diverges from `main`

PR #42's staging database ran migration `005_add_user_roles.sql`. But after PR #43 merged, `main` now has `005_add_audit_log.sql` as migration 5. If PR #42 gets rebased and re-provisioned, the migration runner sees migration 5 as "already applied" (by sequence number) but the content is different.

This produces silent data corruption. The schema doesn't match what the code expects. Queries fail at runtime with column-not-found errors, but only for the staging environment on that specific PR.

### Detection strategy

Before running migrations on a staging environment, I compare the migration files on the branch against what was actually applied in the staging database:

```typescript
interface MigrationRecord {
  sequence: number;
  name: string;
  hash: string; // SHA-256 of the migration file content
  appliedAt: Date;
}

async function detectMigrationDivergence(
  slug: string,
  branchMigrations: string[] // file paths, sorted by sequence
): Promise<DivergenceResult> {
  // Get the migration history from the staging database
  const applied = await getAppliedMigrations(slug);

  const divergences: Divergence[] = [];

  for (const record of applied) {
    const branchFile = branchMigrations.find(
      (f) => extractSequence(f) === record.sequence
    );

    if (!branchFile) {
      // Migration was applied but doesn't exist on the branch anymore
      divergences.push({
        type: 'missing',
        sequence: record.sequence,
        appliedName: record.name,
      });
      continue;
    }

    // Compare the hash of the applied migration with the current file
    const currentHash = hashFile(branchFile);
    if (currentHash !== record.hash) {
      divergences.push({
        type: 'modified',
        sequence: record.sequence,
        appliedName: record.name,
        appliedHash: record.hash,
        currentHash,
      });
    }
  }

  // Check for new migrations that should exist but were skipped
  for (const filePath of branchMigrations) {
    const seq = extractSequence(filePath);
    const wasApplied = applied.some((r) => r.sequence === seq);
    const hasHigherApplied = applied.some((r) => r.sequence > seq);

    if (!wasApplied && hasHigherApplied) {
      divergences.push({
        type: 'gap',
        sequence: seq,
        fileName: path.basename(filePath),
      });
    }
  }

  return {
    diverged: divergences.length > 0,
    divergences,
    recommendation:
      divergences.length > 0 ? 'recreate' : 'migrate-forward',
  };
}
```

The hash comparison is the key insight. Sequence numbers tell you *order*. Hashes tell you *identity*. When migration 5 was applied as `add_user_roles` but the branch now has migration 5 as `add_audit_log`, the sequence matches but the hash doesn't. That's a divergence.

When divergence is detected, the staging system drops the database and re-runs all migrations from scratch. This is the only safe option — you can't reliably "undo" a migration that modified data.

### Storing migration hashes

The migration runner records the SHA-256 hash of each migration file at apply time:

```typescript
async function applyMigration(
  slug: string,
  filePath: string,
  sequence: number
) {
  const content = fs.readFileSync(filePath, 'utf8');
  const hash = crypto
    .createHash('sha256')
    .update(content)
    .digest('hex');

  // Run the migration SQL
  await executeSql(slug, content);

  // Record it in the tracking table
  await executeSql(
    slug,
    `INSERT INTO _staging_migrations (sequence, name, hash, applied_at)
     VALUES ($1, $2, $3, NOW())`,
    [sequence, path.basename(filePath), hash]
  );
}
```

The `_staging_migrations` table is created automatically when the database is provisioned. It's the staging system's table, not the application's. This avoids conflicts with whatever migration framework the application uses (Prisma, Knex, raw SQL).

## Gap-filling port allocation

Each staging environment gets a unique port. The naive approach is to keep a counter and increment it:

```
Port 3001 → PR #42
Port 3002 → PR #43
Port 3003 → PR #44
```

When PR #43 is closed and torn down, port 3002 is freed. But the counter is at 3004 for the next allocation. After a few months, you've allocated ports up to 3150 while only 12 are actually in use. This matters because port ranges are finite and some ranges are reserved by other services.

The gap-filling allocator scans for the lowest available port:

```typescript
interface PortRegistry {
  allocations: Map<string, number>; // slug → port
  range: { min: number; max: number };
}

function allocatePort(registry: PortRegistry): number {
  const usedPorts = new Set(registry.allocations.values());

  for (let port = registry.range.min; port <= registry.range.max; port++) {
    if (!usedPorts.has(port)) {
      return port;
    }
  }

  throw new Error(
    `No available ports in range ${registry.range.min}-${registry.range.max}. ` +
    `${registry.allocations.size} environments active.`
  );
}
```

This is O(n) where n is the range size, but the range is small (typically 3001-3100) and allocation happens once per provisioning run. Micro-optimization here would be premature.

The registry itself is a JSON file protected by the same file-locking mechanism from earlier in this article:

```typescript
async function allocatePortSafely(slug: string): Promise<number> {
  return withLock('port-registry', async () => {
    const registry = loadPortRegistry();

    // Check if this slug already has a port (re-provisioning)
    const existing = registry.allocations.get(slug);
    if (existing !== undefined) {
      return existing;
    }

    const port = allocatePort(registry);
    registry.allocations.set(slug, port);
    savePortRegistry(registry);

    return port;
  });
}
```

Notice the idempotency check — if the slug already has a port (because this is a re-provision after a failed attempt), return the existing allocation instead of grabbing a new one. This prevents port leaks during retry scenarios.

## Putting it all together

These three mechanisms — file locks, migration detection, and port allocation — work together with the saga rollback (Part 1) and YAML AST editing (Part 2) to form a provisioning system that handles concurrency correctly.

The full provisioning flow looks like this:

1. Acquire the port-registry lock, allocate a port, release
2. Acquire the compose lock, add the service via AST, validate, release
3. Create the database (Postgres handles its own concurrency)
4. Detect migration divergence, decide whether to migrate forward or recreate
5. Run migrations, recording hashes
6. Acquire the Caddy lock, add the proxy block, reload, release
7. Start the container, health check

Each lock is held for the minimum time necessary. The compose lock is held during read-modify-validate-write, not during the entire provisioning run. This means two concurrent provisions can interleave: one is pulling its Docker image while the other is editing the compose file.

## Three months in production

Since deploying this system, I've provisioned and torn down several hundred staging environments. The migration divergence detector has caught 8 genuine conflicts that would have produced broken staging environments. The stale lock cleaner has recovered from 3 CI runner crashes without any manual intervention. The port allocator has never leaked a port.

None of this is algorithmically novel. File locks are older than most developers reading this article. Hash-based divergence detection is how Git works. Gap-filling allocation is a freshman CS problem.

The value isn't in the individual pieces. It's in combining them into a system that handles the messy reality of CI-triggered infrastructure provisioning — where everything is concurrent, transient failures are constant, and the cost of getting it wrong is a developer debugging a broken staging environment instead of shipping features.

---

*This wraps up the three-part series. The full staging system is more than these three concerns — there's webhook processing, GitHub status checks, automatic teardown on PR merge, and a monitoring dashboard. But the saga rollback, YAML AST editing, and concurrency controls are the foundation that makes everything else possible.*

*If you're building something similar, start with the saga pattern. Get rollback right first. Everything else is easier when you know failures are handled cleanly.*
