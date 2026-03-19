---
name: gh
description: |
  GitHub CLI and git.
  Use when working with the gh layer.
---

# gh -- GitHub CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `gh`, `git`

## Usage

```yaml
# images.yml or layer.yml
layers:
  - gh
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- GitHub CLI in containers
- Git and gh setup
- The `gh` layer
