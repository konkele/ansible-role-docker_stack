# Ansible Role: Docker Stack Management (Compose & Swarm)

---

## Overview

`ansible-role-docker_stack` is a **declarative, opinionated Ansible role** for managing Docker workloads in **Docker Compose** and **Docker Swarm** environments using a **single, unified data model**.

It is designed for homelabs and production-like clusters where correctness, determinism, and repeatability matter more than convenience shortcuts.

The role is especially useful when:

* The same stack definition must work in both Compose and Swarm
* Secrets must be rotated safely and deterministically
* Services should restart **only** when their inputs actually change
* Validation and normalization should prevent runtime surprises

The role manages the *entire lifecycle* of a Docker stack:

* Variable merging (defaults → group → host → override)
* Validation and fast failure
* Data normalization and preparation
* Directory layout creation
* Network provisioning
* Secret hashing, materialization, and pruning
* Compose file rendering
* Deployment (Compose or Swarm)
* Clean removal with optional pruning

---

## Key Concepts

### Single Canonical Variable: `docker_stack`

All behavior is driven by a single dictionary named `docker_stack`. This structure is built through layered merging and, before any deployment occurs, is:

* Fully merged
* Strictly validated
* Normalized into a canonical schema
* Safe to render directly into Compose v3 syntax

This approach ensures predictable behavior regardless of where values are defined.

### Mode-Aware Behavior

The role adapts automatically based on the selected mode:

| Feature         | Compose          | Swarm                   |
| --------------- | ---------------- | ----------------------- |
| Deployment      | `docker compose` | `docker stack deploy`   |
| Secrets         | Files on disk    | Immutable Swarm secrets |
| Secret rotation | File overwrite   | Hash-suffixed secrets   |
| `deploy:` block | ❌ ignored        | ✅ supported             |
| Networks        | Local / bridge   | Overlay / swarm         |

---

## Features

* Docker Compose **and** Docker Swarm support
* Deterministic, content-addressed secrets (SHA256)
* Automatic service restarts only when secrets change
* Recursive variable merging with predictable precedence
* Strict validation with actionable error messages
* Service normalization (ports, volumes, deploy, secrets)
* External Docker network creation
* Orphaned secret pruning (Compose & Swarm)
* Idempotent, safe stack removal

---

## Variable Model

### Merge Order

The `docker_stack` variable is constructed using a layered dictionary merge:

1. `docker_stack_defaults` (role defaults)
2. `docker_stack_group` (group_vars)
3. `docker_stack_host` (host_vars)
4. `docker_stack_override` (playbook or ad-hoc overrides)

```text
docker_stack = defaults
             + group
             + host
             + override
```

### Merge Semantics

* Dictionaries: recursive merge
* Lists: `append_rp` (append, remove duplicates, preserve order)
* Missing layers: treated as `{}`

This allows safe defaults with precise overrides at any level.

---

## Default Variables

Defined in `defaults/main.yml`:

---

## Stack Lifecycle

### Execution Flow

1. Load optional vars file
2. Merge layered variables
3. Validate input
4. Normalize and prepare data
5. **If `state: absent`** → remove stack
6. **If `state: present`**:

   * Create directories
   * Create networks
   * Materialize secrets
   * Render Compose file
   * Deploy stack
   * Wait for Swarm services (Swarm only)
   * Prune orphaned secrets

Validation always runs, even during removal.

---

## Secrets Model

Secrets are **immutable and content-addressed**.

### How It Works

1. Each secret value is hashed using SHA256
2. Swarm secrets are named:

```text
<secret_name>_<hash[:8]>
```

3. Services reference the hashed secret
4. A combined secret hash is applied as a service label
5. Services restart **only** when referenced secrets change

### Benefits

* No accidental restarts
* Safe secret rotation
* Deterministic deployments
* Easy orphan cleanup

### Compose Mode

* Secrets are written to files under:

```text
<stack>/secrets/<secret_name>
```

* File permissions default to `0600`

---

## Directory Abstractions

The role provides a **directory abstraction layer** that allows services to reference well-known stack directories symbolically, rather than hard-coding absolute paths.

This keeps service definitions:

* Portable between hosts
* Independent of the underlying filesystem layout
* Easier to refactor without touching service definitions

### Available Symbols

The following symbolic prefixes may be used anywhere a host path is expected (most commonly in bind mounts):

| Symbol      | Expands To                            |
| ----------- | ------------------------------------- |
| `$_stack`   | Root directory of the stack           |
| `$_config`  | Stack configuration directory         |
| `$_data`    | Persistent data directory             |
| `$_secrets` | Secrets directory (Compose mode only) |

The actual paths are computed during normalization based on the directory configuration and expanded into concrete filesystem paths before rendering the Compose file.

### Why This Matters

Without abstractions, service definitions often end up tightly coupled to host-specific paths such as `/opt/stacks/foo/data`. Using symbolic directories allows:

* Changing base paths globally without editing services
* Reusing the same service definitions across environments
* Avoiding accidental path drift between Compose and Swarm hosts

### Example: Bind Mount Using Directory Abstractions

```yaml
services:
  app:
    image: ghcr.io/example/app:latest
    volumes:
      - $_config:/etc/app            # bind mount (read-only config)
      - $_data:/var/lib/app          # persistent application data
```

After normalization, this would render to something equivalent to:

```yaml
volumes:
  - /opt/stacks/my_stack/config:/etc/app
  - /opt/stacks/my_stack/data:/var/lib/app
```

The service definition itself never needs to know where the stack actually lives on disk.

---

## Example: Swarm Stack

```yaml
docker_stack_host:
  name: my_swarm_stack
  mode: swarm
  allow_prune: true

  networks:
    proxy:
      external: true
      driver: overlay

  services:
    web:
      image: nginx:latest
      ports:
        - target: 80
          published: 8080
      labels:
        traefik.enable: "true"
      deploy:
        replicas: 3

  secrets:
    db_password:
      value: "{{ vault_db_password }}"
```

---

## Example: Compose Stack

```yaml
docker_stack_host:
  name: my_compose_stack
  mode: compose
  allow_prune: true

  services:
    web:
      image: nginx:latest
      ports:
        - "8080:80"
      volumes:
        - $_config:/etc/nginx/conf.d:ro   # bind mount via directory abstraction
        - $_data:/usr/share/nginx/html    # persistent content directory

  secrets:
    db_password:
      value: "{{ vault_db_password }}"
```

---

## Running the Role

### Deploy

```bash
ansible-playbook site.yml
```

### Remove Stack

```bash
ansible-playbook site.yml --tags remove_stack
```

When removing a stack:

1. Variables are merged
2. Input is validated
3. The stack is stopped and removed
4. Stack directories are deleted **only if** `allow_prune: true`

---

## Design Philosophy

This role is intentionally:

* **Strict** — fail early, fail loudly
* **Predictable** — no implicit magic
* **Idempotent** — no timestamp-based restarts
* **Composable** — works cleanly with Ansible inventory layering

It is well-suited for:

* Homelabs
* Multi-node Swarm clusters
* GitOps-style Ansible repositories
* Long-lived services with rotating secrets

---

## License

MIT License
