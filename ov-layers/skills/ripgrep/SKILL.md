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

## When to Use This Skill

Use when the user asks about:
- ripgrep (rg) in containers
- Fast text search tools
- The `ripgrep` layer
