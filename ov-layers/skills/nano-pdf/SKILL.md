---
name: nano-pdf
description: |
  nano-pdf CLI for PDF editing with natural language.
  Use when working with the nano-pdf layer.
---

# nano-pdf -- PDF editing CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `pixi.toml` |
| Depends | `python` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - nano-pdf
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:python` — required Python runtime dependency
- `/ov-layers:openclaw-full` — metalayer that includes nano-pdf
- `/ov-layers:uv` — sibling Python tooling in openclaw-full

## Related Commands
- `/ov:build` — installs nano-pdf via the pixi builder during image build
- `/ov:shell` — run nano-pdf interactively inside a container

## When to Use This Skill

Use when the user asks about:
- PDF editing with natural language
- nano-pdf CLI tools
- The `nano-pdf` layer
