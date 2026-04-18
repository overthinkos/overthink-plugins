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
# image.yml or layer.yml
layers:
  - mcporter
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:nodejs` — required runtime dependency
- `/ov-layers:openclaw-full` — metalayer that includes mcporter
- `/ov-layers:claude-code` — sibling AI CLI commonly paired with MCP tools

## Related Commands
- `/ov:shell` — run mcporter interactively to list/call MCP tools
- `/ov:build` — installs mcporter via the npm builder during image build

## When to Use This Skill

Use when the user asks about:
- MCP server management in containers
- Listing or calling MCP tools
- The `mcporter` layer
