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
| Install files | `layer.yml`, `user.yml` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - songsee
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:golang` -- build/runtime dependency
- `/ov-layers:openclaw-full` -- parent metalayer that bundles songsee

## Related Commands
- `/ov:shell` -- run songsee CLI inside the container
- `/ov:openclaw` -- audio skill registration in OpenClaw

## When to Use This Skill

Use when the user asks about:
- Audio spectrogram generation
- Audio visualization tools
- The `songsee` layer
