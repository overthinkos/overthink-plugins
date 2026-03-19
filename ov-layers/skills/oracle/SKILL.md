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

## When to Use This Skill

Use when the user asks about:
- Prompt bundling for AI queries
- Multi-engine AI query tools
- The `oracle` layer
