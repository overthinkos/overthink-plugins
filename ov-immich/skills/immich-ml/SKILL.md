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

- `/ov-immich:immich` — Immich server, db init, library/cache volumes
- `/ov-immich:immich-ml` — ML backend for face recognition and smart search
- `/ov-foundation:cuda` — GPU support
- `/ov-foundation:python-ml` — ML Python environment
- `/ov-foundation:postgresql` — database backend
- `/ov-foundation:vectorchord` — VectorChord for smart search
- `/ov-foundation:redis` — session/cache backend

## Related Images

- `/ov-immich:immich` — CPU-only (no ML, no face recognition)
- `/ov-foundation:nvidia` — GPU base without Immich

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
(Fedora 43 rename — see `/ov-foundation:redis`), pytorch + vllm importable
in python-ml pixi env, nodejs24 + cuda, Immich server `dist/main.js`,
DB migrate script, geodata init SQL, ML venv + `immich_ml/` module.
Deploy-scope: port 2283 host-reachable, `/api/server/ping` returns 200
with `pong`, internal ML endpoint reachable via in-container
`curl http://127.0.0.1:3003/ping`. Image-scope: supervisorctl orchestrates
postgresql + redis + immich-server + immich-ml all RUNNING.

## Related Skills

- `/ov-immich:immich`, `/ov-immich:immich-ml`, `/ov-foundation:postgresql`,
  `/ov-foundation:vectorchord`, `/ov-foundation:redis`, `/ov-foundation:nvidia`,
  `/ov-foundation:cuda`, `/ov-foundation:python-ml`, `/ov-coder:nodejs24`,
  `/ov-foundation:supervisord`, `/ov-foundation:dbus`, `/ov-foundation:ov`,
  `/ov-foundation:agent-forwarding`
- `/ov-build:eval` — framework + runtime variable rules (why skips happen)
- `/ov-core:config` — deploy setup (pg password secret, volume backing)
- `/ov-immich:immich` — non-ML variant

## When to Use This Skill

**MUST be invoked** when the task involves the immich-ml image, Immich ML features, or GPU-accelerated photo management. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
