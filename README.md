# Ansible Role: Docker Stack Management (Compose & Swarm)

---

## Overview

`ansible-role-docker_stack` is a **declarative, opinionated Ansible role** for managing Docker workloads in **Docker Compose** and **Docker Swarm** environments using a **single, unified data model**.

It is designed for homelabs and production-like clusters where correctness, determinism, and repeatability matter more than convenience shortcuts.

The role is especially useful when:

* The same stack definition must work in both Compose and Swarm
* Secrets must be rotated safely and deterministically
* Services should restart **only** when their inputs actually change
* Validation and normalization prevent runtime surprises
* Multiple stacks need to be managed from a single playbook

The role manages the *entire lifecycle* of one or more Docker stacks:

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

All behavior is driven by a single dictionary named `docker_stack`. When multiple stacks are defined, the role normalizes them into a list, then processes each stack individually through a consistent workflow.

Before deployment, each `docker_stack` is:

* Fully merged
* Strictly validated
* Normalized into a canonical schema
* Safe to render directly into Compose v3 syntax

This ensures predictable behavior regardless of where values are defined.

### Multi-Stack Support

The role supports multiple stacks in a single playbook by providing `docker_stacks`, which can be defined as either a dictionary or a list of stack definitions. Each stack is processed independently.

```yaml
docker_stacks:
  - name: stack1
    mode: compose
    services:
      ...

  - name: stack2
    mode: swarm
    services:
      ...
```

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
* Multi-stack orchestration from a single playbook

---

## Variable Model

### Merge Order

Each `docker_stack` is constructed using layered merges:

1. `docker_stack_defaults` (role defaults)
2. `docker_stack` (group_vars/host_vars)
3. `docker_stack_override` (playbook or ad-hoc overrides)

```text
docker_stack = defaults
             + group/host
             + override
```

### Merge Semantics

* Dictionaries: recursive merge
* Lists: `append_rp` (append, remove duplicates, preserve order)
* Missing layers: treated as `{}`

The `merge.yml` layer consolidates all per-playbook and ad-hoc overrides, simplifying variable management and ensuring safe, precise overrides at any level.  Same merge strategy is used for `docker-stacks` if utilized.

---

## Stack Lifecycle

### Execution Flow

1. Normalize input stacks (`docker_stacks`) into a consistent list
2. Merge layered variables per stack
3. Validate each stack's input
4. Normalize and prepare data (directories, services, volumes, ports, deploy, secrets)
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

The role provides a **directory abstraction layer** that allows services to reference logical directory names (`base`, `stack`, `config`, `data`, `secrets`) instead of hard-coded paths.

Extra directories can also be defined per stack. All directories have configurable owner, group, and mode.

---

## Services Normalization

The role normalizes all service definitions for:

* Ports (splitting `published:target/proto` strings)
* Volumes (resolving `$_<dir>` syntax to absolute paths)
* Deploy blocks (ensuring placement constraints are always lists)
* Secrets (validating existence and transforming to canonical references)

This ensures a **consistent and predictable Compose/Swarm rendering**.

---

## Docker Networks

External Docker networks defined in `docker_stack.networks` are automatically created if missing, with the correct driver, scope, and attachable flags.

* Compose mode: `bridge`
* Swarm mode: `overlay`

---

## Orphaned Secret Pruning

Secrets not referenced by any service are pruned:

* Compose mode: deleted from disk
* Swarm mode: removed using Docker secret API

This ensures a clean, predictable environment.

---

## Deployment

Stacks are deployed according to their `mode`:

* Compose mode: `docker compose up -d`
* Swarm mode: `docker stack deploy`

Automatic restarts happen **only** if secrets or other inputs have changed.

---

## Usage Example

```yaml
- hosts: all
  roles:
    - role: docker_stack
      vars:
        docker_stacks:
          - name: webapp
            mode: compose
            services:
              app:
                image: myapp:latest
                ports:
                  - "8080:80"
                secrets:
                  - source: app_secret
            secrets:
              app_secret:
                value: supersecret

          - name: db
            mode: swarm
            services:
              postgres:
                image: postgres:15
                environment:
                  POSTGRES_PASSWORD: mypassword
```

This will deploy two independent stacks with their respective mode, directories, and secrets fully normalized and managed.

---

## License

MIT License
