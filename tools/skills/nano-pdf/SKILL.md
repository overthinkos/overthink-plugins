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
| Install files | `charly.yml`, `pixi.toml` |
| Depends | `python` |

## Usage

```yaml
# box or candy charly.yml
layers:
  - nano-pdf
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-languages:python` — required Python runtime dependency
- `/charly-openclaw:openclaw-full` — metalayer that includes nano-pdf
- `/charly-coder:uv` — sibling Python tooling in openclaw-full

## Related Commands
- `/charly-build:build` — installs nano-pdf via the pixi builder during image build
- `/charly-core:shell` — run nano-pdf interactively inside a container

## When to Use This Skill

Use when the user asks about:
- PDF editing with natural language
- nano-pdf CLI tools
- The `nano-pdf` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
