---
name: immich
description: |
  Immich photo management server on port 2283 with PostgreSQL and Redis.
  Use when working with Immich, photo management, or media library services.
---

# immich -- Self-hosted photo management server

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord`, `nodejs24`, `postgresql`, `redis` |
| Ports | 2283 |
| Volumes | `library` -> `~/.immich/library`, `cache` -> `~/.immich/cache` |
| Service | `immich-db-init` (oneshot, priority 20), `immich-server` (priority 30) |
| Route | `immich.localhost:2283` |
| Install files | `root.yml`, `user.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `IMMICH_VERSION` | `v2.5.6` |
| `IMMICH_HOST` / `IMMICH_PORT` | `0.0.0.0` / `2283` |
| `IMMICH_MEDIA_LOCATION` | `~/.immich/library` |
| `IMMICH_MACHINE_LEARNING_ENABLED` | `false` |
| `DB_HOSTNAME` / `DB_DATABASE_NAME` | `127.0.0.1` / `immich` |
| `DB_USERNAME` / `DB_PASSWORD` | `immich` / `immich` |
| `REDIS_HOSTNAME` / `REDIS_PORT` | `127.0.0.1` / `6379` |
| `NODE_ENV` | `production` |

## Packages

- `vips`, `vips-devel`, `ffmpeg`, `perl-Image-ExifTool` (RPM)
- `libheif`, `LibRaw`, `gcc-c++`, `make`, `unzip` (RPM)
- Fedora Multimedia repo (negativo17)

## Usage

```yaml
# images.yml
immich:
  layers:
    - immich
```

## Used In Images

- `/ov-images:immich`
- `/ov-images:immich-cuda`

## Related Layers

- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:nodejs24` -- Node.js runtime dependency
- `/ov-layers:postgresql` -- database dependency
- `/ov-layers:redis` -- cache dependency
- `/ov-layers:immich-ml` -- optional ML backend

## When to Use This Skill

Use when the user asks about:

- Immich photo management setup
- Photo library configuration
- Immich database or Redis connectivity
- Port 2283 or `immich.localhost` route
- Media processing (vips, ffmpeg, ExifTool)
