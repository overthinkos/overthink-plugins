---
name: wacli
description: |
  WhatsApp CLI for message sending and history sync.
  Use when working with the wacli layer.
---

# wacli -- WhatsApp CLI

## Layer Properties

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
  - wacli
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:golang` — Go toolchain dependency
- `/charly-openclaw:openclaw-full` — metalayer that bundles wacli
- `/charly-hermes:hermes` — companion messaging stack with WhatsApp bridge

## Related Commands
- `/charly-core:shell` — run wacli inside the container
- `/charly-build:secrets` — store WhatsApp session credentials

## When to Use This Skill

Use when the user asks about:
- WhatsApp messaging from containers
- WhatsApp history sync
- The `wacli` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
