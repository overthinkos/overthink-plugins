---
name: ffmpeg
description: |
  FFmpeg multimedia framework (negativo17 nonfree build with H.264/AAC support).
  Use when working with the ffmpeg layer.
---

# ffmpeg -- FFmpeg multimedia

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |
| Repo | negativo17 `fedora-multimedia` |

## Packages

RPM: `ffmpeg` (from negativo17 `fedora-multimedia` repo — full nonfree build with H.264, AAC, etc.)

**Note:** This is NOT `ffmpeg-free` (RPM Fusion). This is the negativo17 build which includes nonfree codecs required for H.264 encoding/decoding.

## Usage

```yaml
# images.yml or layer.yml
layers:
  - ffmpeg
```

All layers that need ffmpeg should declare it as a dependency rather than independently adding the negativo17 repo. This ensures a single authoritative install point.

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)
- `hermes` (via `hermes` layer `depends: ffmpeg`)
- `hermes-playwright` (via `hermes` layer `depends: ffmpeg`)
- `immich` (via `immich` layer `depends: ffmpeg`)
- `immich-ml` (via `immich` layer `depends: ffmpeg`)
- CUDA-based images (via `cuda` layer `depends: ffmpeg`)
- Whisper-based images (via `whisper` layer `depends: ffmpeg`)

## When to Use This Skill

Use when the user asks about:
- FFmpeg in containers
- Multimedia processing tools
- The `ffmpeg` layer
