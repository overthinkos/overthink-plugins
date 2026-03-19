---
name: immich-cuda
description: |
  Immich photo management with CUDA ML backend for face recognition
  and smart search. Includes PostgreSQL, Redis, and the immich-ml service.
  Use when working with Immich ML features or the GPU-accelerated deployment.
---

# immich-cuda

Immich photo management with GPU-accelerated machine learning for face recognition and smart search.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | nodejs24, cuda, python-ml, supervisord, postgresql, redis, immich, immich-ml |
| Platforms | linux/amd64 |
| Ports | 2283 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `pixi` → `python` → `supervisord` (transitive)
3. `nodejs24` — Node.js 24 runtime
4. `cuda` — CUDA toolkit, cuDNN
5. `python-ml` — ML Python environment
6. `postgresql` — database on :5432
7. `redis` — cache on :6379
8. `immich` — Immich server on :2283
9. `immich-ml` — ML backend on :3003

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
| models | ~/.immich/models | ML models |

## Quick Start

```bash
ov build immich-cuda
ov start immich-cuda
# Open http://localhost:2283
```

## Key Layers

- `/ov-layers:immich` — Immich server, db init, library/cache volumes
- `/ov-layers:immich-ml` — ML backend for face recognition and smart search
- `/ov-layers:cuda` — GPU support
- `/ov-layers:python-ml` — ML Python environment
- `/ov-layers:postgresql` — database backend
- `/ov-layers:redis` — session/cache backend

## Related Images

- `/ov-images:immich` — CPU-only (no ML, no face recognition)
- `/ov-images:nvidia` — GPU base without Immich

## When to Use This Skill

Use when the user asks about Immich with ML, face recognition, smart search, the immich-cuda image, or GPU-accelerated photo management.
