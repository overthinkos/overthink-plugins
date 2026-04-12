---
name: ripgrep
description: |
  Fast recursive text search (rg).
  Use when working with the ripgrep layer.
---

# ripgrep -- Fast text search

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `ripgrep`

## Usage

```yaml
# images.yml or layer.yml
layers:
  - ripgrep
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:dev-tools` -- bundles ripgrep alongside other CLI utilities
- `/ov-layers:openclaw-full` -- parent metalayer that bundles ripgrep

## Related Commands
- `/ov:shell` -- run rg interactively inside the container
- `/ov:cmd` -- one-shot rg invocation in a running service

## When to Use This Skill

Use when the user asks about:
- ripgrep (rg) in containers
- Fast text search tools
- The `ripgrep` layer
