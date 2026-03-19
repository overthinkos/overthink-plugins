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

## When to Use This Skill

Use when the user asks about:
- uv Python package manager
- Fast Python dependency management
- The `uv` layer
