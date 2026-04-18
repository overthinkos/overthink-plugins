---
name: redis
description: |
  Redis in-memory data store on port 6379 with periodic persistence.
  Use when working with Redis, caching, or session storage in containers.
---

# redis -- Redis in-memory data store

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | none |
| Ports | 6379 |
| Service | `redis` (supervisord, priority 20) |
| Install files | `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `REDIS_URL` | `redis://127.0.0.1:6379` |

## Service Environment (injected into other containers)

| Variable | Template Value | Resolved Example |
|----------|---------------|-----------------|
| `REDIS_URL` | `redis://{{.ContainerName}}:6379` | `redis://ov-redis:6379` |

Pod-aware: same-container consumers receive `redis://localhost:6379`, cross-container consumers receive `redis://ov-redis:6379`. When `ov config` runs, `REDIS_URL` is automatically injected into the global `deploy.yml` env for Redis service discovery.

See `/ov:layer` for `env_provides` field docs.

## Packages

- `redis` (RPM)

## Usage

```yaml
# image.yml -- typically used as dependency of immich
my-image:
  layers:
    - redis
```

## Used In Images

- `/ov-images:immich`
- `/ov-images:immich-ml`

## Related Layers

- `/ov-layers:immich` -- primary consumer (depends on redis)
- `/ov-layers:postgresql` -- often paired with redis in service stacks

## When to Use This Skill

Use when the user asks about:

- Redis setup in containers
- Caching or session storage
- Port 6379 service
- `REDIS_URL` configuration
