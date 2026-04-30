---
name: claude-code
description: |
  Claude Code CLI installed globally via npm from @anthropic-ai/claude-code.
  Use when working with Claude Code, AI coding assistants, or Anthropic tooling.
---

# claude-code -- Claude Code CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs` |
| Install files | `layer.yml`, `tasks:` |

PATH additions: `~/.local/bin`

## Usage

```yaml
# image.yml
my-dev:
  layers:
    - claude-code
```

## Used In Images

- No enabled images use this layer directly (standalone tool layer)

## Related Skills

- `/ov-coder:nodejs` -- required dependency (provides npm)
- `/ov-coder:codex`, `/ov-coder:gemini` — sibling AI CLIs (all three share the npm-global install pattern)
- `/ov-hermes:hermes-full-layer` — metalayer that bundles this with codex, gemini, dev-tools, devops-tools
- `/ov-hermes:hermes` — primary image that ships this CLI
- `/ov-build:layer` — layer authoring reference
- `/ov-build:eval` — declarative testing framework (this layer's test verifies `${HOME}/.npm-global/bin/claude` + `claude --version`)

## When to Use This Skill

Use when the user asks about:

- Claude Code CLI in containers
- Anthropic AI coding tools
- The `claude-code` layer
