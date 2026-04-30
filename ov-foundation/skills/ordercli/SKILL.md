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
- `/ov-coder:golang` -- build/runtime dependency
- `/ov-openclaw:openclaw-full` -- parent metalayer that bundles ordercli

## Related Commands
- `/ov-core:shell` -- run ordercli inside the container
- `/ov-advanced:openclaw` -- skill registration in OpenClaw gateway

## When to Use This Skill

Use when the user asks about:
- Food delivery order tracking
- Foodora order status
- The `ordercli` layer

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
