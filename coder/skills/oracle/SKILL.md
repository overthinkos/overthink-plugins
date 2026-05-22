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
# image.yml or layer.yml
layers:
  - oracle
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:nodejs` -- runtime dependency
- `/ov-openclaw:openclaw-full` -- parent metalayer that bundles oracle

## Related Commands
- `/ov-automation:openclaw-deploy` -- gateway/skill configuration
- `/ov-core:shell` -- run oracle CLI inside the container

## When to Use This Skill

Use when the user asks about:
- Prompt bundling for AI queries
- Multi-engine AI query tools
- The `oracle` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
