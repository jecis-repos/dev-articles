---
title: "Build Your Own Staging System, Part 2: YAML AST Editing with Validation"
published: false
description: "Stop using regex to edit docker-compose.yml. Use the YAML AST to add services, modify ports, and preserve comments — with automated validation and backup recovery."
tags: devops, typescript, yaml, docker
series: "Build Your Own Staging System"
cover_image:
canonical_url:
---

In [Part 1](https://dev.to/jecis/build-your-own-staging-system-part-1-saga-pattern-for-infrastructure-provisioning), I covered using the Saga pattern to roll back failed provisioning steps. One of those steps is "add a service block to `docker-compose.yml`." Sounds simple. It nearly broke me.

My staging system manages a single `docker-compose.yml` that can have 15+ service definitions at any given time — one per open pull request. Services get added, modified, and removed constantly. The compose file is the source of truth for what's running on the staging server.

The first version used string concatenation. The second version used regex. The third version — the one that's been running in production for three months — uses YAML AST editing. Here's why, and how.

## The regex graveyard

Let me show you the regex approach that "worked" for two weeks:

```typescript
function addServiceToCompose(slug: string, config: ServiceConfig) {
  let content = fs.readFileSync('docker-compose.yml', 'utf8');
  const serviceBlock = `
  ${slug}:
    image: ${config.image}
    ports:
      - "${config.port}:3000"
    env_file:
      - ./${slug}/.env
    depends_on:
      - postgres
`;
  // Find the services: key and append after the last service
  content = content.replace(
    /(services:[\s\S]*?)(\nvolumes:)/,
    `$1${serviceBlock}$2`
  );
  fs.writeFileSync('docker-compose.yml', content);
}
```

This fails when:
- There's no `volumes:` key at the end (the regex anchor is gone)
- A comment contains the word "volumes:" 
- The indentation isn't exactly 2 spaces
- There are YAML anchors or aliases
- Someone manually edits the file and adds a trailing newline
- The file uses CRLF line endings (CI runner on a different OS)

Each failure mode produced a subtly broken compose file. Docker Compose would either refuse to parse it (best case) or silently misinterpret the structure (worst case — a service inheriting another service's port mapping because the indentation was off by one space).

## Enter the YAML AST

The `yaml` npm package (not `js-yaml` — this is important) provides document-level APIs that parse YAML into an Abstract Syntax Tree and can write it back out while preserving comments, key ordering, and formatting.

```bash
npm install yaml
```

The key difference from `js-yaml`: the `yaml` package gives you `parseDocument()`, which returns a `Document` object with `YAMLMap` and `YAMLSeq` nodes you can manipulate directly. `js-yaml` gives you a plain JavaScript object — all comments, ordering, and formatting are lost.

### Reading the document

```typescript
import { parseDocument, YAMLMap, Pair, Scalar } from 'yaml';
import fs from 'fs';

function loadComposeDocument() {
  const raw = fs.readFileSync('docker-compose.yml', 'utf8');
  return parseDocument(raw);
}
```

`parseDocument` gives you a live AST. You can read it, modify it, and call `toString()` to get back valid YAML — with your original comments and formatting intact.

### Adding a service

Here's the real implementation for adding a service to the compose file:

```typescript
function addService(slug: string, config: ServiceConfig) {
  const doc = loadComposeDocument();

  // Navigate to the services map
  const services = doc.get('services') as YAMLMap;
  if (!services) {
    throw new Error('docker-compose.yml has no services key');
  }

  // Check if service already exists (idempotency)
  if (services.has(slug)) {
    log.warn(`Service ${slug} already exists in compose file`);
    return;
  }

  // Build the service definition as a nested structure
  const serviceDefinition = doc.createNode({
    image: config.image,
    container_name: `staging-${slug}`,
    ports: [`${config.port}:3000`],
    env_file: [`./${slug}/.env`],
    depends_on: ['postgres'],
    networks: ['staging-net'],
    restart: 'unless-stopped',
    labels: {
      'staging.slug': slug,
      'staging.pr': String(config.prNumber),
      'staging.created': new Date().toISOString(),
    },
  });

  // Add to the services map
  services.add(new Pair(new Scalar(slug), serviceDefinition));

  // Write back
  fs.writeFileSync('docker-compose.yml', doc.toString());
}
```

What this gives you over string manipulation:

1. **Correct indentation automatically.** The AST knows the document's indentation style and applies it consistently.
2. **No injection vulnerabilities.** If a branch name contains a colon or a hash, string interpolation produces invalid YAML. The AST escapes values correctly.
3. **Idempotency for free.** `services.has(slug)` checks the parsed structure, not a regex match that might find the slug inside a comment.

### Adding a volume

Same pattern, different node:

```typescript
function addVolume(slug: string) {
  const doc = loadComposeDocument();

  let volumes = doc.get('volumes') as YAMLMap;
  if (!volumes) {
    // Create the volumes key if it doesn't exist
    doc.add(new Pair(new Scalar('volumes'), doc.createNode({})));
    volumes = doc.get('volumes') as YAMLMap;
  }

  const volumeName = `staging-${slug}-data`;
  if (!volumes.has(volumeName)) {
    volumes.add(
      new Pair(
        new Scalar(volumeName),
        doc.createNode({ driver: 'local' })
      )
    );
  }

  fs.writeFileSync('docker-compose.yml', doc.toString());
}
```

### Modifying port mappings

When a provisioning run needs to assign a new port (because the original was taken), you modify the existing service:

```typescript
function updateServicePort(slug: string, newPort: number) {
  const doc = loadComposeDocument();
  const services = doc.get('services') as YAMLMap;

  const service = services.get(slug) as YAMLMap;
  if (!service) {
    throw new Error(`Service ${slug} not found in compose file`);
  }

  // Replace the ports array
  service.set('ports', doc.createNode([`${newPort}:3000`]));

  fs.writeFileSync('docker-compose.yml', doc.toString());
}
```

### Removing a service

The teardown path, used during rollback (Part 1) and when a PR is merged or closed:

```typescript
function removeService(slug: string) {
  const doc = loadComposeDocument();
  const services = doc.get('services') as YAMLMap;

  if (!services.has(slug)) {
    log.warn(`Service ${slug} not found in compose file, nothing to remove`);
    return;
  }

  services.delete(slug);

  // Also remove the associated volume if it exists
  const volumes = doc.get('volumes') as YAMLMap;
  const volumeName = `staging-${slug}-data`;
  if (volumes?.has(volumeName)) {
    volumes.delete(volumeName);
  }

  fs.writeFileSync('docker-compose.yml', doc.toString());
}
```

## Post-write validation

Editing the AST correctly is necessary but not sufficient. Docker Compose has its own validation rules that go beyond valid YAML — port conflicts, unknown keys, invalid image references. After every write, I validate:

```typescript
async function writeAndValidateCompose(doc: Document) {
  const content = doc.toString();
  const bakPath = `docker-compose.yml.bak-${Date.now()}`;

  // Backup the current file
  fs.copyFileSync('docker-compose.yml', bakPath);

  // Write the new content
  fs.writeFileSync('docker-compose.yml', content);

  // Validate with Docker Compose itself
  try {
    await execAsync('docker compose config --quiet', {
      timeout: 10_000,
    });
    // Validation passed — remove the backup
    fs.unlinkSync(bakPath);
  } catch (error) {
    // Validation failed — restore from backup
    log.error('Compose validation failed, restoring backup', error);
    fs.copyFileSync(bakPath, 'docker-compose.yml');
    fs.unlinkSync(bakPath);
    throw new Error(`Compose file invalid after edit: ${error.message}`);
  }
}
```

The `docker compose config --quiet` command parses the compose file, resolves all references, and exits with a non-zero code if anything is wrong. The `--quiet` flag suppresses the resolved output (which you don't need — you just want the validation).

The timestamped `.bak` file is important. If two provisioning runs overlap (more on this in Part 3), you don't want them overwriting each other's backups. Each backup has a unique name, and the run that created it is responsible for cleaning it up.

## Why this matters more than you think

I want to be direct about why I'm spending an entire article on YAML editing. It's because **config file mutation is the most dangerous operation in infrastructure automation**, and most systems treat it as trivial.

Here's what happens in practice: you write a regex-based compose editor. It works for months. Then someone names their branch `feature/update-volumes-api`. Your regex matches "volumes" in the branch name and corrupts the file. Or a YAML comment gets mis-parsed and a service inherits the wrong network. Or a race condition between two CI runners produces a file with duplicate keys that Docker Compose silently resolves by taking the last one.

These bugs are:
- **Rare** — they happen once every few hundred operations
- **Silent** — the system doesn't crash, it just behaves wrong
- **Destructive** — by the time you notice, multiple environments are affected
- **Hard to reproduce** — they depend on the exact content of the file at the time of the edit

AST-based editing eliminates the entire category. The parser understands YAML structure. The serializer produces valid YAML. The validation step catches anything the AST can't (Docker-specific rules). The backup provides a recovery path even if all of that fails.

## The comment preservation test

Here's a concrete example of why `yaml` (AST) beats `js-yaml` (parse-to-object). Given this input:

```yaml
services:
  # Core infrastructure
  postgres:
    image: postgres:16
    # Keep data across restarts
    volumes:
      - pg-data:/var/lib/postgresql/data
  
  # Staging environments below
  pr-142:
    image: myapp:pr-142
    ports:
      - "3001:3000"
```

After adding a new service with the AST approach and calling `doc.toString()`:

```yaml
services:
  # Core infrastructure
  postgres:
    image: postgres:16
    # Keep data across restarts
    volumes:
      - pg-data:/var/lib/postgresql/data
  
  # Staging environments below
  pr-142:
    image: myapp:pr-142
    ports:
      - "3001:3000"
  pr-157:
    image: myapp:pr-157
    ports:
      - "3002:3000"
    env_file:
      - ./pr-157/.env
```

Comments preserved. Ordering preserved. Blank lines preserved. The `js-yaml` round-trip would strip every comment and potentially reorder the keys.

When you're debugging at 2am and you open the compose file, those comments are the difference between understanding the file in 5 seconds and staring at it for 5 minutes.

## Practical tips

A few things I learned the hard way:

**Always re-read the file before editing.** Don't cache the parsed document across operations. Another process might have modified the file between your read and write. Read, parse, modify, validate, write — every time.

**Use `doc.createNode()` for nested structures.** Don't try to build `YAMLMap` and `YAMLSeq` objects manually. `createNode` handles the conversion from plain JS objects and arrays, including proper scalar types.

**Test with adversarial branch names.** Names like `fix/yaml: broken`, `feature/#123`, and `release/v2.0.0-rc.1` all contain characters that are meaningful in YAML. The AST handles them. Regex doesn't.

**Keep the backup rotation bounded.** I keep the last 5 backups and delete older ones. Disk is cheap, but a directory with 3,000 `.bak` files makes `ls` annoying.

---

*In Part 3, we'll tackle the concurrency problem head-on: POSIX file locks to prevent two CI runners from editing the compose file simultaneously, and a migration detection system that catches database divergence before it causes data corruption.*
