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
| Install files | `candy.yml`, `pixi.toml` |
| Depends | `python` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - nano-pdf
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-languages:python` — required Python runtime dependency
- `/ov-openclaw:openclaw-full` — metalayer that includes nano-pdf
- `/ov-coder:uv` — sibling Python tooling in openclaw-full

## Related Commands
- `/ov-build:build` — installs nano-pdf via the pixi builder during image build
- `/ov-core:shell` — run nano-pdf interactively inside a container

## When to Use This Skill

Use when the user asks about:
- PDF editing with natural language
- nano-pdf CLI tools
- The `nano-pdf` layer

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
