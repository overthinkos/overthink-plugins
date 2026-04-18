---
name: immich-ml
description: |
  Immich photo management with CUDA ML backend for face recognition
  and smart search. Includes PostgreSQL, Redis, and the immich-ml service.
  MUST be invoked before building, deploying, configuring, or troubleshooting the immich-ml image.
---

# immich-ml

Immich photo management with GPU-accelerated machine learning for face recognition and smart search.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, nodejs24, cuda, python-ml, supervisord, postgresql, vectorchord, redis, immich, immich-ml |
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
7. `vectorchord` — VectorChord vector similarity extension
8. `redis` — cache on :6379
9. `immich` — Immich server on :2283
10. `immich-ml` — ML backend on :3003

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 2283 | Immich web UI + API | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| library | ~/.immich/library | Photo/video storage |
| cache | ~/.immich/cache | Thumbnail cache |
| import | ~/.immich/import | Photo import directory |
| external | ~/.immich/external | External library (no-copy) |
| pgdata | ~/.postgresql | PostgreSQL data |
| models | ~/.immich/models | ML models |

## Quick Start

```bash
ov image build immich-ml
ov config setup immich-ml
ov start immich-ml
# Open http://localhost:2283
```

## Key Layers

- `/ov-layers:immich` — Immich server, db init, library/cache volumes
- `/ov-layers:immich-ml` — ML backend for face recognition and smart search
- `/ov-layers:cuda` — GPU support
- `/ov-layers:python-ml` — ML Python environment
- `/ov-layers:postgresql` — database backend
- `/ov-layers:vectorchord` — VectorChord for smart search
- `/ov-layers:redis` — session/cache backend

## Related Images

- `/ov-images:immich` — CPU-only (no ML, no face recognition)
- `/ov-images:nvidia` — GPU base without Immich

## Verification

After `ov start`:
- `ov status immich-ml` — container running
- `ov service status immich-ml` — all services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:2283` — Immich HTTP returns 200
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:3003` — ML backend HTTP returns 200

## Test Coverage

Latest `ov test immich-ml` run: **61 passed, 0 failed, 2 skipped**.
The 2 skips are `redis-responds` and `redis-port-open` — they reference
`${HOST_PORT:6379}` which isn't mapped on this image (redis is internal
to the pod). Correct skip behavior; no authoring action needed.

Covers postgres binaries + pg_isready, `valkey-compat-redis` package
(Fedora 43 rename — see `/ov-layers:redis`), pytorch + vllm importable
in python-ml pixi env, nodejs24 + cuda, Immich server `dist/main.js`,
DB migrate script, geodata init SQL, ML venv + `immich_ml/` module.
Deploy-scope: port 2283 host-reachable, `/api/server/ping` returns 200
with `pong`, internal ML endpoint reachable via in-container
`curl http://127.0.0.1:3003/ping`. Image-scope: supervisorctl orchestrates
postgresql + redis + immich-server + immich-ml all RUNNING.

## Related Skills

- `/ov-layers:immich`, `/ov-layers:immich-ml`, `/ov-layers:postgresql`,
  `/ov-layers:vectorchord`, `/ov-layers:redis`, `/ov-layers:cuda`,
  `/ov-layers:python-ml`, `/ov-layers:nodejs24`
- `/ov:test` — framework + runtime variable rules (why skips happen)
- `/ov:config` — deploy setup (pg password secret, volume backing)
- `/ov-images:immich` — non-ML variant

## When to Use This Skill

**MUST be invoked** when the task involves the immich-ml image, Immich ML features, or GPU-accelerated photo management. Invoke this skill BEFORE reading source code or launching Explore agents.
