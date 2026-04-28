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
| Install files | `layer.yml`, `tasks:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - wacli
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:golang` — Go toolchain dependency
- `/ov-layers:openclaw-full` — metalayer that bundles wacli
- `/ov-layers:hermes` — companion messaging stack with WhatsApp bridge

## Related Commands
- `/ov:shell` — run wacli inside the container
- `/ov:secrets` — store WhatsApp session credentials

## When to Use This Skill

Use when the user asks about:
- WhatsApp messaging from containers
- WhatsApp history sync
- The `wacli` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
