---
name: uv
description: |
  uv Python package manager.
  Use when working with the uv layer.
---

# uv -- Python package manager

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `pixi.toml` |
| Depends | `python` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - uv
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:python` — Python runtime dependency
- `/ov-layers:pixi` — alternate Python package manager
- `/ov-layers:openclaw-full` — metalayer that bundles uv

## Related Commands
- `/ov:shell` — run uv inside the container
- `/ov:build` — rebuild after layer changes

## When to Use This Skill

Use when the user asks about:
- uv Python package manager
- Fast Python dependency management
- The `uv` layer
