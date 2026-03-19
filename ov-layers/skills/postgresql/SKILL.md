---
name: postgresql
description: |
  PostgreSQL database server on port 5432 with pgvector extension and persistent data.
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
| Install files | `root.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `PGDATA` | `~/.postgresql/data` |
| `POSTGRES_DB` | `immich` |
| `POSTGRES_USER` | `immich` |
| `POSTGRES_PASSWORD` | `immich` |

## Packages

- `postgresql-server` (RPM)
- `postgresql-contrib` (RPM)
- `pgvector` (RPM) -- vector similarity search extension

## Usage

```yaml
# images.yml -- typically used as dependency of immich
my-image:
  layers:
    - postgresql
```

## Used In Images

- `/ov-images:immich`
- `/ov-images:immich-cuda`

## Related Layers

- `/ov-layers:immich` -- primary consumer (depends on postgresql)
- `/ov-layers:redis` -- often paired with postgresql in service stacks

## When to Use This Skill

Use when the user asks about:

- PostgreSQL database setup in containers
- pgvector extension
- Database volume or `PGDATA` configuration
- Port 5432 service
- Database initialization
