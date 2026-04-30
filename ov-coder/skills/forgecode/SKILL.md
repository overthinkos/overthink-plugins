---
name: forgecode
description: |
  Forge AI coding agent CLI (forgecode.dev) installed globally via npm from
  the `forgecode` package. Installs the `forge` binary.
  Use when working with Forge, forgecode.dev, or alternative AI coding agents.
---

# forgecode -- Forge AI Coding Agent CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs` |
| Install files | `layer.yml`, `package.json` |
| npm package | `forgecode` |
| Installed binary | `forge` (at `${HOME}/.npm-global/bin/forge`) |

## Usage

```yaml
# image.yml
my-dev:
  layers:
    - forgecode
```

## Used In Images

- `hermes-full` (metalayer) -- bundles forgecode alongside claude-code / codex / gemini
- `hermes` image -- ships forgecode transitively via hermes-full

## Related Skills

- `/ov-coder:nodejs` -- required dependency (provides npm)
- `/ov-coder:claude-code`, `/ov-coder:codex`, `/ov-coder:gemini` -- sibling AI CLIs (same npm-global install pattern)
- `/ov-hermes:hermes-full-layer` -- metalayer that bundles this CLI
- `/ov-hermes:hermes` -- primary image that ships this CLI
- `/ov-build:layer` -- layer authoring reference
- `/ov-build:eval` -- declarative testing framework (this layer verifies `${HOME}/.npm-global/bin/forge` + `forge --version`)

## When to Use This Skill

Use when the user asks about:

- Forge / forgecode / forgecode.dev in containers
- Alternative AI coding agents alongside Claude Code / Codex / Gemini
- The `forgecode` layer
