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
layers:
  - songsee
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:golang` -- build/runtime dependency
- `/charly-openclaw:openclaw-full` -- parent metalayer that bundles songsee

## Related Commands
- `/charly-core:shell` -- run songsee CLI inside the container
- `/charly-automation:openclaw-deploy` -- audio skill registration in OpenClaw

## When to Use This Skill

Use when the user asks about:
- Audio spectrogram generation
- Audio visualization tools
- The `songsee` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
