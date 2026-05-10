---
name: testapi
description: |
  FastAPI test service on port 9090 routed via testapi.localhost for development testing.
  Use when working with the test API, Traefik routing validation, or service health checks.
---

# testapi -- FastAPI test service

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Ports | 9090 |
| Route | `testapi.localhost:9090` |
| Service | `testapi` (supervisord) |
| Install files | `task:`, `pixi.toml`, `app.py` |

## Usage

```yaml
# image.yml
fedora-test:
  layers:
    - testapi
```

## Used In Images

- `/ov-distros:fedora-test` (disabled)

## Related Layers

- `/ov-infrastructure:supervisord` -- process manager dependency
- `/ov-infrastructure:traefik` -- reverse proxy for route handling

## When to Use This Skill

Use when the user asks about:

- Test API service
- Traefik route testing
- `testapi.localhost` endpoint
- Port 9090 service

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
