---
name: traefik
description: |
  Traefik reverse proxy on ports 8000/8080/443 with automatic TLS and dynamic routing.
  Use when working with Traefik, reverse proxy, TLS certificates, or service routing.
---

# traefik -- Reverse proxy with dynamic routing

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Ports | 8000, 8080, 443 |
| Volumes | `certs` -> `~/.traefik/acme` |
| Service | `traefik` (supervisord) |
| Install files | `tasks:`, `traefik.yml` |

## Usage

```yaml
# image.yml
fedora-test:
  layers:
    - traefik
```

## Used In Images

- `/ov-images:fedora-test` (disabled)

## Related Layers

- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:testapi` -- test service with Traefik route

## Related Commands

- `/ov:list` — `ov image list routes` shows Traefik route configuration

## When to Use This Skill

Use when the user asks about:

- Traefik reverse proxy setup
- Dynamic routing configuration
- TLS certificate management (ACME)
- Ports 8000, 8080, or 443
- Service routing via `*.localhost`
