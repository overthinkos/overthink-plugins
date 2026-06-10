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
| Install files | `charly.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - codex
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:nodejs` — Required Node.js runtime parent dependency
- `/charly-coder:claude-code` — Sibling AI coding agent CLI bundled in same metalayers
- `/charly-coder:gemini` — Sibling Google Gemini CLI bundled alongside

## Related Commands
- `/charly-build:build` — Builds the layer (npm global install via package.json)
- `/charly-core:shell` — Interactive shell to invoke `codex` inside the container

## When to Use This Skill

Use when the user asks about:
- OpenAI Codex CLI in containers
- AI coding agent setup
- The `codex` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
