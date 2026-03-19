---
name: ffmpeg
description: |
  FFmpeg multimedia framework.
  Use when working with the ffmpeg layer.
---

# ffmpeg -- FFmpeg multimedia

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `ffmpeg-free`

## Usage

```yaml
# images.yml or layer.yml
layers:
  - ffmpeg
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- FFmpeg in containers
- Multimedia processing tools
- The `ffmpeg` layer
