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
| Install files | `layer.yml`, `pixi.toml` |
| Depends | `python`, `cuda`, `ffmpeg` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - whisper
```

## Used In Images

- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers

- `/ov-foundation:python` — Python runtime — required dependency
- `/ov-foundation:cuda` — CUDA toolkit — required dependency
- `/ov-selkies:ffmpeg` — FFmpeg multimedia (nonfree codecs) — required dependency
- `/ov-foundation:sherpa-onnx` — alternative STT engine (ONNX-based, lighter weight)

## When to Use This Skill

Use when the user asks about:
- Speech-to-text in containers
- OpenAI Whisper setup
- Audio transcription with CUDA
- The `whisper` layer

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
