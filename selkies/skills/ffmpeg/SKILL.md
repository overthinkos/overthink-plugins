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
| Install files | `candy.yml` (packages only) |
| Depends | none |
| Repo | negativo17 `fedora-multimedia` |

## Packages

RPM: `ffmpeg` (from negativo17 `fedora-multimedia` repo — full nonfree build with H.264, AAC, etc.)

**Note:** This is NOT `ffmpeg-free` (RPM Fusion). This is the negativo17 build which includes nonfree codecs required for H.264 encoding/decoding.

## Usage

```yaml
# box.yml or candy.yml
layers:
  - ffmpeg
```

All layers that need ffmpeg should declare it as a dependency rather than independently adding the negativo17 repo. This ensures a single authoritative install point.

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `hermes` (via `hermes` layer `requires: ffmpeg`)
- `hermes-playwright` (via `hermes` layer `requires: ffmpeg`)
- `immich` (via `immich` layer `requires: ffmpeg`)
- `immich-ml` (via `immich` layer `requires: ffmpeg`)
- CUDA-based images (via `cuda` layer `requires: ffmpeg`)
- Whisper-based images (via `whisper` layer `requires: ffmpeg`)

## Related Layers
- `/charly-distros:cuda` — Depends on ffmpeg for GPU video processing
- `/charly-tools:whisper` — Depends on ffmpeg for audio transcoding
- `/charly-hermes:hermes` — Depends on ffmpeg for media tooling
- `/charly-immich:immich` — Depends on ffmpeg for photo/video transcoding

## Related Commands
- `/charly-build:build` — Builds the ffmpeg negativo17 RPM into the image
- `/charly-core:shell` — Interactive shell to run `ffmpeg` for media processing

## When to Use This Skill

Use when the user asks about:
- FFmpeg in containers
- Multimedia processing tools
- The `ffmpeg` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
