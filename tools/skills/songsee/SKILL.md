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
| Install files | `candy.yml`, `task:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - songsee
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:golang` -- build/runtime dependency
- `/ov-openclaw:openclaw-full` -- parent metalayer that bundles songsee

## Related Commands
- `/ov-core:shell` -- run songsee CLI inside the container
- `/ov-automation:openclaw-deploy` -- audio skill registration in OpenClaw

## When to Use This Skill

Use when the user asks about:
- Audio spectrogram generation
- Audio visualization tools
- The `songsee` layer

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
