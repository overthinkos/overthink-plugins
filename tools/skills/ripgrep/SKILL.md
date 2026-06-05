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
| Install files | `candy.yml` (packages only) |
| Depends | none |

## Packages

RPM: `ripgrep`

## Usage

```yaml
# box.yml or candy.yml
layers:
  - ripgrep
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:dev-tools` -- bundles ripgrep alongside other CLI utilities
- `/ov-openclaw:openclaw-full` -- parent metalayer that bundles ripgrep

## Related Commands
- `/ov-core:shell` -- run rg interactively inside the container
- `/ov-core:cmd` -- one-shot rg invocation in a running service

## When to Use This Skill

Use when the user asks about:
- ripgrep (rg) in containers
- Fast text search tools
- The `ripgrep` layer

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
