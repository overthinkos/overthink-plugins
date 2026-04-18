---
name: ordercli
description: |
  Food delivery order status CLI (Foodora).
  Use when working with the ordercli layer.
---

# ordercli -- Food delivery order CLI

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
  - ordercli
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:golang` -- build/runtime dependency
- `/ov-layers:openclaw-full` -- parent metalayer that bundles ordercli

## Related Commands
- `/ov:shell` -- run ordercli inside the container
- `/ov:openclaw` -- skill registration in OpenClaw gateway

## When to Use This Skill

Use when the user asks about:
- Food delivery order tracking
- Foodora order status
- The `ordercli` layer
