---
name: codex
description: |
  OpenAI Codex CLI coding agent.
  Use when working with the codex layer.
---

# codex -- OpenAI Codex CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - codex
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- OpenAI Codex CLI in containers
- AI coding agent setup
- The `codex` layer
