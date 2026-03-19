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
| Depends | `python`, `cuda` |

## Packages

RPM: `ffmpeg-free`

## Usage

```yaml
# images.yml or layer.yml
layers:
  - whisper
```

## Used In Images

- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- Speech-to-text in containers
- OpenAI Whisper setup
- Audio transcription with CUDA
- The `whisper` layer
