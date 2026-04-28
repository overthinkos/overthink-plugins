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

- `redis` (dnf request) → installed as **`valkey-compat-redis`** on
  Fedora 43. The old `redis` package was replaced; dnf resolves the
  name via `Provides:`, but `rpm -q redis` returns "not installed".
  Any `package:` test must query the real installed name. See
  `/ov:eval` Authoring Gotcha #8.

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

## Tests

The layer ships 5 declarative checks embedded in the `org.overthinkos.eval`
OCI label (see `/ov:eval` for the full schema — this layer is the
**gold-standard pattern** referenced there):

- **Build-scope** (run under `ov eval image`, via `podman run --rm`):
  - `redis-binary` — `/usr/bin/redis-server` exists
  - `redis-cli-binary` — `/usr/bin/redis-cli` exists
  - `redis-package` — `valkey-compat-redis` package installed (real
    installed provider on Fedora 43; see Packages note above)
- **Deploy-scope** (run under `ov test` against a live service; uses
  `${HOST_PORT:6379}` runtime substitution so the checks keep working
  if `deploy.yml` remaps the host port — host-side tests always use
  `127.0.0.1:${HOST_PORT:N}`, not `${CONTAINER_IP}`):
  - `redis-responds` — `redis-cli -h 127.0.0.1 -p ${HOST_PORT:6379} ping` returns `PONG`
  - `redis-port-open` — TCP dial `127.0.0.1:${HOST_PORT:6379}` succeeds

**Composed-vs-standalone skip behavior**: when redis is a sub-layer of
a larger image (e.g. `immich-ml`) that doesn't expose port 6379 on the
host, the deploy-scope tests correctly skip with
`unresolved variables: HOST_PORT:6379`. No authoring action needed.

## Related Skills

- `/ov-layers:immich` -- primary consumer (depends on redis)
- `/ov-layers:postgresql` -- often paired with redis in service stacks
- `/ov:eval` -- declarative testing framework (this layer is the gold-standard pattern)
- `/ov:layer` -- layer authoring + `env_provides` field docs
- `/ov-layers:valkey` -- Remi-repo Valkey 9 package (separate layer; different version than Fedora's default valkey-compat-redis)

## When to Use This Skill

Use when the user asks about:

- Redis setup in containers
- Caching or session storage
- Port 6379 service
- `REDIS_URL` configuration
