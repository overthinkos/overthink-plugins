---
name: gemini
description: |
  Google Gemini CLI for AI coding assistance and search.
  Use when working with the gemini layer.
---

# gemini -- Google Gemini CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - gemini
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:nodejs` — required runtime dependency
- `/ov-layers:claude-code` — sibling AI CLI in openclaw-full and hermes-full
- `/ov-layers:codex` — sibling AI CLI in openclaw-full and hermes-full

## Related Commands
- `/ov:secrets` — provision Gemini API credentials for the CLI
- `/ov:shell` — run gemini interactively inside a container
- `/ov:build` — installs gemini during image build

## When to Use This Skill

Use when the user asks about:
- Google Gemini CLI in containers
- AI coding assistance tools
- The `gemini` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
