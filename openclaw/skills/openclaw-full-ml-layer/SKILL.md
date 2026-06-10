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
| Install files | `charly.yml` (metalayer, layers only) |
| Depends | none (composes other layers) |

## Composed Layers

This metalayer extends `openclaw-full` with ML capabilities:

- `openclaw-full` -- all OpenClaw tools (25 layers)
- `whisper` -- OpenAI Whisper speech-to-text (requires CUDA)
- `sherpa-onnx` -- Offline text-to-speech

## Usage

```yaml
# charly.yml
candy:
```

## Used In Images


## Related Layers
- `/charly-openclaw:openclaw-full` -- base composition with all OpenClaw tools
- `/charly-tools:whisper` -- speech-to-text component
- `/charly-tools:sherpa-onnx` -- offline TTS component

## Related Commands
- `/charly-automation:openclaw-deploy` -- gateway configuration and channel setup
- `/charly-build:build` -- build images composing this metalayer

## When to Use This Skill

Use when the user asks about:
- OpenClaw with ML/AI capabilities
- Speech-to-text and text-to-speech in OpenClaw
- CUDA-accelerated OpenClaw deployment
- The `openclaw-full-ml` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
