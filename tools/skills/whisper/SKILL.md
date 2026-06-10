---
name: whisper
description: |
  OpenAI Whisper local speech-to-text.
  Use when working with the whisper layer.
---

# whisper -- OpenAI Whisper STT

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `pixi.toml` |
| Depends | `python`, `cuda`, `ffmpeg` |

## Usage

```yaml
# box or candy charly.yml
layers:
  - whisper
```

## Used In Images


## Related Layers

- `/charly-languages:python` — Python runtime — required dependency
- `/charly-distros:cuda` — CUDA toolkit — required dependency
- `/charly-selkies:ffmpeg` — FFmpeg multimedia (nonfree codecs) — required dependency
- `/charly-tools:sherpa-onnx` — alternative STT engine (ONNX-based, lighter weight)

## When to Use This Skill

Use when the user asks about:
- Speech-to-text in containers
- OpenAI Whisper setup
- Audio transcription with CUDA
- The `whisper` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
