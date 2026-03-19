---
name: mcporter
description: |
  MCP server CLI for listing, configuring, and calling MCP tools.
  Use when working with the mcporter layer.
---

# mcporter -- MCP server CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - mcporter
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- MCP server management in containers
- Listing or calling MCP tools
- The `mcporter` layer
