---
name: oracle
description: |
  Oracle CLI for prompt bundling and multi-engine AI queries.
  Use when working with the oracle layer.
---

# oracle -- Prompt bundling CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - oracle
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:nodejs` -- runtime dependency
- `/ov-layers:openclaw-full` -- parent metalayer that bundles oracle

## Related Commands
- `/ov:openclaw` -- gateway/skill configuration
- `/ov:shell` -- run oracle CLI inside the container

## When to Use This Skill

Use when the user asks about:
- Prompt bundling for AI queries
- Multi-engine AI query tools
- The `oracle` layer
