---
name: songsee
description: |
  Audio spectrogram and visualization CLI.
  Use when working with the songsee layer.
---

# songsee -- Audio spectrogram CLI

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
  - songsee
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-coder:golang` -- build/runtime dependency
- `/ov-openclaw:openclaw-full` -- parent metalayer that bundles songsee

## Related Commands
- `/ov-core:shell` -- run songsee CLI inside the container
- `/ov-advanced:openclaw` -- audio skill registration in OpenClaw

## When to Use This Skill

Use when the user asks about:
- Audio spectrogram generation
- Audio visualization tools
- The `songsee` layer

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
