---
name: supervisord
description: |
  Supervisord process manager for running multiple services inside containers.
  Use when working with supervisord, container service management, or multi-process containers.
---

# supervisord -- process manager

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `python` |
| Install files | `layer.yml` |

## Packages

- `supervisor` (RPM) -- process control system

## Usage

```yaml
# images.yml
my-image:
  layers:
    - supervisord
    - my-service  # layers with service: entries need supervisord
```

## Used In Images

Transitive dependency for all images with managed services, including:
`openclaw`, `openclaw-sway-browser`, `openclaw-ollama-sway-browser`, `openclaw-ollama`, `jupyter`, `ollama`, `comfyui`, `immich`, `immich-cuda`

## Related Layers

- `/ov-layers:python` -- Python runtime dependency
- `/ov-layers:traefik` -- reverse proxy (depends on supervisord)
- `/ov-layers:dbus` -- D-Bus session bus (depends on supervisord)
- `/ov-layers:ollama` -- LLM server (depends on supervisord)
- `/ov-layers:openclaw` -- AI gateway (depends on supervisord)
- `/ov-layers:postgresql` -- database (service layer)
- `/ov-layers:redis` -- cache (service layer)

## When to Use This Skill

Use when the user asks about:

- Supervisord configuration or process management
- Running multiple services in a container
- Service startup order or priorities
- The `ov service` commands (start/stop/restart/status)
- `supervisor` RPM package
