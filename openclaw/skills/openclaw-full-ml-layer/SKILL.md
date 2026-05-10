---
name: openclaw-full-ml-layer
description: |
  OpenClaw full + ML tools (whisper, sherpa-onnx-tts, CUDA).
  Use when working with the openclaw-full-ml layer.
---

# openclaw-full-ml -- OpenClaw full + ML tools

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (metalayer, layers only) |
| Depends | none (composes other layers) |

## Composed Layers

This metalayer extends `openclaw-full` with ML capabilities:

- `openclaw-full` -- all OpenClaw tools (25 layers)
- `whisper` -- OpenAI Whisper speech-to-text (requires CUDA)
- `sherpa-onnx` -- Offline text-to-speech

## Usage

```yaml
# image.yml
layers:
  - openclaw-full-ml
```

## Used In Images

- `openclaw-full-ml` (with `sway-desktop` and `ollama`)

## Related Layers
- `/ov-openclaw:openclaw-full` -- base composition with all OpenClaw tools
- `/ov-tools:whisper` -- speech-to-text component
- `/ov-tools:sherpa-onnx` -- offline TTS component

## Related Commands
- `/ov-automation:openclaw-deploy` -- gateway configuration and channel setup
- `/ov-build:build` -- build images composing this metalayer

## When to Use This Skill

Use when the user asks about:
- OpenClaw with ML/AI capabilities
- Speech-to-text and text-to-speech in OpenClaw
- CUDA-accelerated OpenClaw deployment
- The `openclaw-full-ml` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
