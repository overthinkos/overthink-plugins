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
| Install files | `charly.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box or candy charly.yml
layers:
  - gemini
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:nodejs` — required runtime dependency
- `/charly-coder:claude-code` — sibling AI CLI in openclaw-full and hermes-full
- `/charly-coder:codex` — sibling AI CLI in openclaw-full and hermes-full

## Related Commands
- `/charly-build:secrets` — provision Gemini API credentials for the CLI
- `/charly-core:shell` — run gemini interactively inside a container
- `/charly-build:build` — installs gemini during image build

## When to Use This Skill

Use when the user asks about:
- Google Gemini CLI in containers
- AI coding assistance tools
- The `gemini` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
