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
| Install files | `user.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `REDIS_URL` | `redis://127.0.0.1:6379` |

## Packages

- `redis` (RPM)

## Usage

```yaml
# images.yml -- typically used as dependency of immich
my-image:
  layers:
    - redis
```

## Used In Images

- `/ov-images:immich`
- `/ov-images:immich-cuda`

## Related Layers

- `/ov-layers:immich` -- primary consumer (depends on redis)
- `/ov-layers:postgresql` -- often paired with redis in service stacks

## When to Use This Skill

Use when the user asks about:

- Redis setup in containers
- Caching or session storage
- Port 6379 service
- `REDIS_URL` configuration
