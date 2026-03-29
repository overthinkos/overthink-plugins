---
name: wl-record-pixelflux
description: |
  Desktop video recorder via pixelflux H.264 pipeline.
  Use when working with the wl-record-pixelflux layer.
---

# wl-record-pixelflux -- Desktop video recording via pixelflux pipeline

## Overview

Provides `pixelflux-record` for recording desktop video on selkies-desktop. Taps directly into pixelflux's `ScreenCapture` API — the same rendering pipeline that powers the WebSocket video stream and screenshots.

**Primary mode (H.264):** Uses `output_mode=1` to receive pre-encoded H.264 frames from pixelflux, piped to ffmpeg for MP4 container muxing only (no re-encoding, near-zero CPU overhead).

**Fallback mode (JPEG):** If H.264 output mode is unavailable, falls back to `output_mode=0` (JPEG stripes), stitches frames with Pillow, and pipes to ffmpeg for H.264 encoding.

**Why not wf-recorder?** labwc running nested inside pixelflux advertises `wlr-screencopy` but can't deliver frames because pixelflux owns the render target. `pixelflux-record` bypasses this by capturing from the rendering pipeline directly (same reason `pixelflux-screenshot` exists for screenshots).

## Layer Definition

```yaml
depends:
  - selkies
  - ffmpeg
```

Uses pixelflux and Pillow from the selkies layer's pixi environment. Requires ffmpeg for MP4 muxing/encoding.

## How It Works

### H.264 Mode (Primary)

1. Creates `ScreenCapture` with `output_mode=1`, `h264_streaming_mode=True`, `h264_fullframe=True`
2. Callback receives H.264 NAL units directly from pixelflux's encoder
3. Writes raw H.264 data to ffmpeg stdin pipe
4. ffmpeg muxes to MP4: `ffmpeg -f h264 -c:v copy output.mp4` (no transcode)
5. Optional audio: `-f pulse -i default.monitor -c:a aac`

### JPEG Fallback Mode

1. Creates `ScreenCapture` with `output_mode=0` (JPEG stripes)
2. Collects stripes per frame, stitches with Pillow (same as pixelflux-screenshot)
3. Writes full JPEG frames to ffmpeg stdin pipe
4. ffmpeg re-encodes: `ffmpeg -f image2pipe -c:v mjpeg -c:v libx264 output.mp4`

### Signal Handling

SIGINT (Ctrl-C) or SIGTERM triggers graceful shutdown: stops capture, closes ffmpeg pipe, waits for MP4 finalization.

## Key Properties

| Property | Value |
|----------|-------|
| Depends | `selkies` (pixelflux + Pillow), `ffmpeg` (muxing/encoding) |
| Install | `~/.local/bin/pixelflux-record` (Python script) |
| Protocol | pixelflux `ScreenCapture` API (H.264 or JPEG mode) |
| Output | MP4 (H.264 video + optional AAC audio) |
| Audio | PulseAudio monitor source via ffmpeg |

## CLI Usage

```bash
# Direct usage
pixelflux-record output.mp4                  # 30fps, video only
pixelflux-record output.mp4 --fps 60         # 60fps
pixelflux-record output.mp4 --audio          # video + audio
pixelflux-record output.mp4 --fps 60 --audio # 60fps + audio
# Stop with Ctrl-C
```

## Integration with `ov record`

```bash
# Start desktop video recording (auto-detects pixelflux-record)
ov record start selkies-desktop -n demo --mode desktop --audio

# Run visible terminal commands
ov record term selkies-desktop "echo hello" -n demo

# Interact with desktop
ov cdp open selkies-desktop "https://example.com"
ov wl click selkies-desktop 640 360

# Stop and copy to host
ov record stop selkies-desktop -n demo -o demo.mp4
```

## Included In

- `selkies-desktop` metalayer

## Cross-References

- `/ov:record` — `ov record start --mode desktop` auto-detects pixelflux-record
- `/ov-layers:wl-screenshot-pixelflux` — Screenshot companion (same API, JPEG mode only)
- `/ov-layers:wf-recorder` — Alternative for sway-desktop (wlr-screencopy)
- `/ov-layers:selkies` — Parent layer (provides pixelflux + Pillow)
- `/ov-layers:ffmpeg` — Required dependency (MP4 muxing/encoding)

## When to Use This Skill

Use when the user asks about:
- Desktop video recording on selkies-desktop
- The pixelflux recording pipeline
- The `wl-record-pixelflux` layer
