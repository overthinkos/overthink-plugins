---
name: sherpa-onnx
description: |
  sherpa-onnx offline text-to-speech.
  Use when working with the sherpa-onnx layer.
---

# sherpa-onnx -- Offline TTS

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `tasks:` |
| Depends | none |

## Environment

| Variable | Value |
|----------|-------|
| `SHERPA_ONNX_RUNTIME_DIR` | `~/.local/share/sherpa-onnx/runtime` |
| `SHERPA_ONNX_MODEL_DIR` | `~/.local/share/sherpa-onnx/models` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - sherpa-onnx
```

## Used In Images

- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-openclaw:openclaw-full-ml` -- parent metalayer that bundles sherpa-onnx
- `/ov-foundation:whisper` -- speech-to-text counterpart
- `/ov-foundation:sag` -- ElevenLabs TTS alternative

## Related Commands
- `/ov-core:shell` -- run sherpa-onnx CLI inside the container
- `/ov-advanced:openclaw` -- TTS skill registration in OpenClaw

## When to Use This Skill

Use when the user asks about:
- Offline text-to-speech
- sherpa-onnx TTS engine
- ONNX-based speech synthesis
- The `sherpa-onnx` layer

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
