---
name: camsnap
description: |
  RTSP/ONVIF camera snapshot and clip CLI.
  Use when working with the camsnap layer.
---

# camsnap -- Camera snapshot CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `task:` |
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
  - camsnap
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:golang` — Required Go runtime parent dependency
- `/ov-openclaw:openclaw-full` — Metalayer that bundles camsnap with other agent CLIs

## Related Commands
- `/ov-build:build` — Builds the layer (Go install via a cmd task)
- `/ov-core:shell` — Interactive shell to run camsnap inside the container

## When to Use This Skill

Use when the user asks about:
- RTSP or ONVIF camera snapshots
- Camera snapshot/clip tools
- The `camsnap` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
