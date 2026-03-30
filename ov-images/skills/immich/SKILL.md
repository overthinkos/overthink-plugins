---
name: immich
description: |
  Immich photo management server on port 2283. Includes PostgreSQL, Redis,
  and non-free codec support via RPM Fusion. CPU-only (no ML).
  MUST be invoked before building, deploying, configuring, or troubleshooting the immich image.
---

# immich

Self-hosted photo and video management server with full codec support.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora-nonfree |
| Layers | nodejs24, supervisord, postgresql, redis, immich |
| Platforms | linux/amd64 |
| Ports | 2283 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `fedora-nonfree` (RPM Fusion for codecs)
2. `pixi` → `python` → `supervisord` (transitive)
3. `nodejs24` — Node.js 24 runtime
4. `postgresql` — database on :5432
5. `redis` — cache on :6379
6. `immich` — Immich server on :2283, library + cache volumes

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 2283 | Immich web UI + API | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| library | ~/.immich/library | Photo/video storage |
| cache | ~/.immich/cache | Thumbnail cache |
| pgdata | ~/.postgresql | PostgreSQL data |

## Quick Start

```bash
ov build immich
ov config immich
ov start immich
# Open http://localhost:2283
```

## Key Layers

- `/ov-layers:immich` — Immich server, db init, library/cache volumes
- `/ov-layers:postgresql` — database backend
- `/ov-layers:redis` — session/cache backend
- `/ov-layers:rpmfusion` — non-free codec support (via fedora-nonfree base)

## Related Images

- `/ov-images:fedora-nonfree` — parent base
- `/ov-images:immich-cuda` — adds CUDA ML for face recognition and smart search

## Verification

After `ov start`:
- `ov status immich` — container running
- `ov service status immich` — all services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:2283` — Immich HTTP returns 200

## When to Use This Skill

**MUST be invoked** when the task involves the immich image, photo management, or the CPU-only Immich setup. Invoke this skill BEFORE reading source code or launching Explore agents.
