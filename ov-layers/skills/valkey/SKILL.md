---
name: valkey
description: |
  Valkey 9.x key-value store (Redis-compatible) on port 6379 via Remi modular repo.
  Use when working with Valkey, Redis-compatible caching, or the valkey layer.
---

# valkey -- Valkey key-value store

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | none |
| Ports | `6379` |
| Service | `valkey` (supervisord, priority 20) |
| Volume | `valkey-data` -> `~/.valkey` |
| Install files | `layer.yml` only |

## Environment Variables

| Variable | Value |
|----------|-------|
| `VALKEY_URL` | `redis://127.0.0.1:6379` |

## Packages

- `valkey` (RPM, from Remi modular repo, module `valkey:remi-9.0`)

## Usage

```yaml
# images.yml
my-app:
  layers:
    - valkey
  ports:
    - "6379:6379"
```

## Related Layers

- `/ov-layers:redis` -- Redis alternative (same port 6379)

## When to Use This Skill

Use when the user asks about:

- Valkey key-value store
- Redis-compatible caching in containers
- Port 6379 services
