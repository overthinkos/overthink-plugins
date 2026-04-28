---
name: postgresql
description: |
  PostgreSQL database server on port 5432 with pgvector extension and persistent data.
  Entrypoint supports POSTGRES_SHARED_PRELOAD_LIBRARIES for extension loading.
  Use when working with PostgreSQL, database configuration, or pgvector.
---

# postgresql -- PostgreSQL database server

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | none |
| Ports | 5432 |
| Volumes | `pgdata` -> `~/.postgresql/data` |
| Service | `postgresql` (supervisord, priority 10) |
| Install files | `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `PGDATA` | `~/.postgresql/data` |
| `POSTGRES_HOST_AUTH_METHOD` | `scram-sha-256` |
| `POSTGRES_SHARED_PRELOAD_LIBRARIES` | *(optional)* — comma-separated `.so` names loaded at startup |

The entrypoint also reads these variables (with defaults, not set in layer.yml):
- `POSTGRES_USER` (default: `postgres`) — set by consuming layers (e.g., immich sets `immich`)
- `POSTGRES_DB` (default: `$POSTGRES_USER`) — set by consuming layers
- `POSTGRES_PASSWORD` — required unless `POSTGRES_HOST_AUTH_METHOD=trust`

## Service Environment (injected into other containers)

| Variable | Template Value | Resolved Example |
|----------|---------------|-----------------|
| `PGHOST` | `{{.ContainerName}}` | `ov-postgresql` |
| `PGPORT` | `5432` | `5432` |

Pod-aware: same-container consumers receive `PGHOST=localhost`, cross-container consumers receive `PGHOST=ov-postgresql`. When `ov config` runs, these are automatically injected into the global `deploy.yml` env for PostgreSQL service discovery.

See `/ov:layer` for `env_provides` field docs.

## Packages

- `postgresql-server` (RPM)
- `postgresql-contrib` (RPM)
- `pgvector` (RPM) -- vector similarity search extension

## Usage

```yaml
# image.yml -- typically used as dependency of immich
my-image:
  layers:
    - postgresql
```

## Used In Images

- `/ov-images:immich`
- `/ov-images:immich-ml`

## Entrypoint Features

The custom entrypoint (`/usr/local/bin/postgresql-entrypoint.sh`) supports:

- **First-run initialization** — `initdb`, database creation, and `/docker-entrypoint-initdb.d/` scripts
- **Password management** — `POSTGRES_PASSWORD` or `POSTGRES_PASSWORD_FILE`
- **`POSTGRES_SHARED_PRELOAD_LIBRARIES`** — when set, adds `-c shared_preload_libraries=...` to both the init-phase temp server and the final exec. Used by the `vectorchord` layer to load `vchord.so`. Generic mechanism — any extension layer can use it.

## Related Layers

- `/ov-layers:immich` -- primary consumer (depends on postgresql)
- `/ov-layers:vectorchord` -- sets `POSTGRES_SHARED_PRELOAD_LIBRARIES=vchord.so`
- `/ov-layers:redis` -- often paired with postgresql in service stacks

## Related Commands

- `/ov:config` — Deploy with secrets provisioning (db-password)
- `/ov:secrets` — Manage database credentials

## When to Use This Skill

Use when the user asks about:

- PostgreSQL database setup in containers
- pgvector extension
- Database volume or `PGDATA` configuration
- Port 5432 service
- Database initialization

## Related

- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
