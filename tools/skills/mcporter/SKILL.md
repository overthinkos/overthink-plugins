---
name: mcporter
description: |
  MCP server CLI for listing, configuring, and calling MCP tools.
  Use when working with the mcporter candy.
---

# mcporter -- MCP server CLI

## Candy Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - mcporter
```

## Used In Boxes

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Candies
- `/charly-coder:nodejs` — required runtime dependency
- `/charly-openclaw:openclaw-full` — metalayer that includes mcporter
- `/charly-coder:claude-code` — sibling AI CLI commonly paired with MCP tools

## Related Commands
- `/charly-core:shell` — run mcporter interactively to list/call MCP tools
- `/charly-build:build` — installs mcporter via the npm builder during image build

## When to Use This Skill

Use when the user asks about:
- MCP server management in containers
- Listing or calling MCP tools
- The `mcporter` candy

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
