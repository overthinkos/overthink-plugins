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
| Install files | `candy.yml`, `task:` |
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
# box.yml or candy.yml
layers:
  - sag
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:golang` -- build/runtime dependency
- `/charly-tools:sherpa-onnx` -- offline TTS sibling
- `/charly-tools:whisper` -- speech-to-text counterpart

## Related Commands
- `/charly-core:shell` -- run sag CLI inside the container
- `/charly-build:secrets` -- supply ELEVENLABS_API_KEY

## When to Use This Skill

Use when the user asks about:
- ElevenLabs text-to-speech
- TTS in containers
- The `sag` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
