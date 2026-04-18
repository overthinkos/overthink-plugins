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
# images.yml or layer.yml
layers:
  - sag
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:golang` -- build/runtime dependency
- `/ov-layers:sherpa-onnx` -- offline TTS sibling
- `/ov-layers:whisper` -- speech-to-text counterpart

## Related Commands
- `/ov:shell` -- run sag CLI inside the container
- `/ov:secrets` -- supply ELEVENLABS_API_KEY

## When to Use This Skill

Use when the user asks about:
- ElevenLabs text-to-speech
- TTS in containers
- The `sag` layer
