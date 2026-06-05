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
| Install files | `candy.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - gemini
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:nodejs` — required runtime dependency
- `/ov-coder:claude-code` — sibling AI CLI in openclaw-full and hermes-full
- `/ov-coder:codex` — sibling AI CLI in openclaw-full and hermes-full

## Related Commands
- `/ov-build:secrets` — provision Gemini API credentials for the CLI
- `/ov-core:shell` — run gemini interactively inside a container
- `/ov-build:build` — installs gemini during image build

## When to Use This Skill

Use when the user asks about:
- Google Gemini CLI in containers
- AI coding assistance tools
- The `gemini` layer

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
