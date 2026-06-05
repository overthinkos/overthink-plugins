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
| Install files | `candy.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - mcporter
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:nodejs` — required runtime dependency
- `/ov-openclaw:openclaw-full` — metalayer that includes mcporter
- `/ov-coder:claude-code` — sibling AI CLI commonly paired with MCP tools

## Related Commands
- `/ov-core:shell` — run mcporter interactively to list/call MCP tools
- `/ov-build:build` — installs mcporter via the npm builder during image build

## When to Use This Skill

Use when the user asks about:
- MCP server management in containers
- Listing or calling MCP tools
- The `mcporter` layer

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
