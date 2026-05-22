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
# image.yml or layer.yml
layers:
  - codex
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:nodejs` — Required Node.js runtime parent dependency
- `/ov-coder:claude-code` — Sibling AI coding agent CLI bundled in same metalayers
- `/ov-coder:gemini` — Sibling Google Gemini CLI bundled alongside

## Related Commands
- `/ov-build:build` — Builds the layer (npm global install via package.json)
- `/ov-core:shell` — Interactive shell to invoke `codex` inside the container

## When to Use This Skill

Use when the user asks about:
- OpenAI Codex CLI in containers
- AI coding agent setup
- The `codex` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
