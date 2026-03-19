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
  - camsnap
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- RTSP or ONVIF camera snapshots
- Camera snapshot/clip tools
- The `camsnap` layer
