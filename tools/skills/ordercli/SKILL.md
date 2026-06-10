---
name: ordercli
description: |
  Food delivery order status CLI (Foodora).
  Use when working with the ordercli candy.
---

# ordercli -- Food delivery order CLI

## Candy Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `task:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - ordercli
```

## Used In Boxes

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Candies
- `/charly-coder:golang` -- build/runtime dependency
- `/charly-openclaw:openclaw-full` -- parent metalayer that bundles ordercli

## Related Commands
- `/charly-core:shell` -- run ordercli inside the container
- `/charly-automation:openclaw-deploy` -- skill registration in OpenClaw gateway

## When to Use This Skill

Use when the user asks about:
- Food delivery order tracking
- Foodora order status
- The `ordercli` candy

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
