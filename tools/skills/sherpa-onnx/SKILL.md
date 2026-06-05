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
| Install files | `candy.yml`, `task:` |
| Depends | none |

## Environment

| Variable | Value |
|----------|-------|
| `SHERPA_ONNX_RUNTIME_DIR` | `~/.local/share/sherpa-onnx/runtime` |
| `SHERPA_ONNX_MODEL_DIR` | `~/.local/share/sherpa-onnx/models` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - sherpa-onnx
```

## Used In Images


## Related Layers
- `/ov-tools:whisper` -- speech-to-text counterpart
- `/ov-tools:sag` -- ElevenLabs TTS alternative

## Related Commands
- `/ov-core:shell` -- run sherpa-onnx CLI inside the container
- `/ov-automation:openclaw-deploy` -- TTS skill registration in OpenClaw

## When to Use This Skill

Use when the user asks about:
- Offline text-to-speech
- sherpa-onnx TTS engine
- ONNX-based speech synthesis
- The `sherpa-onnx` layer

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
