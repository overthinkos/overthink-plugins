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
| Install files | `layer.yml`, `root.yml` |

PATH additions: `~/.local/bin`

## Usage

```yaml
# images.yml
my-dev:
  layers:
    - claude-code
```

## Used In Images

- No enabled images use this layer directly (standalone tool layer)

## Related Layers

- `/overthink-layers:nodejs` -- required dependency (provides npm)

## When to Use This Skill

Use when the user asks about:

- Claude Code CLI in containers
- Anthropic AI coding tools
- The `claude-code` layer
