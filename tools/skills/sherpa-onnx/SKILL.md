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
| Install files | `charly.yml`, `task:` |
| Depends | none |

## Environment

| Variable | Value |
|----------|-------|
| `SHERPA_ONNX_RUNTIME_DIR` | `~/.local/share/sherpa-onnx/runtime` |
| `SHERPA_ONNX_MODEL_DIR` | `~/.local/share/sherpa-onnx/models` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - sherpa-onnx
```

## Used In Images


## Related Layers
- `/charly-tools:whisper` -- speech-to-text counterpart
- `/charly-tools:sag` -- ElevenLabs TTS alternative

## Related Commands
- `/charly-core:shell` -- run sherpa-onnx CLI inside the container
- `/charly-automation:openclaw-deploy` -- TTS skill registration in OpenClaw

## When to Use This Skill

Use when the user asks about:
- Offline text-to-speech
- sherpa-onnx TTS engine
- ONNX-based speech synthesis
- The `sherpa-onnx` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
