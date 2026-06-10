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
| Install files | `charly.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - oracle
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:nodejs` -- runtime dependency
- `/charly-openclaw:openclaw-full` -- parent metalayer that bundles oracle

## Related Commands
- `/charly-automation:openclaw-deploy` -- gateway/skill configuration
- `/charly-core:shell` -- run oracle CLI inside the container

## When to Use This Skill

Use when the user asks about:
- Prompt bundling for AI queries
- Multi-engine AI query tools
- The `oracle` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
