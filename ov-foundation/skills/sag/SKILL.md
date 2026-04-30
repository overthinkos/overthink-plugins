---
name: sag
description: |
  ElevenLabs text-to-speech CLI.
  Use when working with the sag layer.
---

# sag -- ElevenLabs TTS CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `tasks:` |
| Depends | `golang` |

## Packages

RPM: `alsa-lib-devel`

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - sag
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-coder:golang` -- build/runtime dependency
- `/ov-foundation:sherpa-onnx` -- offline TTS sibling
- `/ov-foundation:whisper` -- speech-to-text counterpart

## Related Commands
- `/ov-core:shell` -- run sag CLI inside the container
- `/ov-build:secrets` -- supply ELEVENLABS_API_KEY

## When to Use This Skill

Use when the user asks about:
- ElevenLabs text-to-speech
- TTS in containers
- The `sag` layer

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
