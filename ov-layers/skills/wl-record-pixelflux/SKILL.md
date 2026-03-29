---
name: wl-record-pixelflux
description: |
  Desktop video recorder via selkies WebSocket capture bridge for selkies-desktop.
  Use when working with the wl-record-pixelflux layer.
---

# wl-record-pixelflux -- Desktop video recording via selkies capture bridge

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `user.yml`, `pixelflux-record` (Python script) |
| Depends | `selkies` (capture bridge + WebSocket stream), `ffmpeg` (MP4 muxing) |

## What It Does

Provides `pixelflux-record` for recording desktop video on selkies-desktop. Connects to the capture bridge at `/tmp/ov-capture.sock` which relays the same H.264 frames that the browser sees via the selkies WebSocket stream. No direct ScreenCapture API access -- taps into the existing selkies streaming pipeline to avoid creating a second capture instance (the Rust backend only supports one active capture at a time).

**Why not wf-recorder?** labwc running nested inside pixelflux can't deliver wlr-screencopy frames. The capture bridge bypasses this by tapping into the selkies WebSocket stream directly.

## How It Works

1. Connects to Unix socket at `/tmp/ov-capture.sock` (started by `selkies-capture-server`)
2. Sends `STREAM\n` to request continuous H.264 frame data
3. Receives frames as `4-byte length + raw H.264 data` pairs
4. Pipes raw H.264 data to ffmpeg with `-use_wallclock_as_timestamps 1`
5. ffmpeg re-encodes to MP4 with correct wall-clock timing
6. SIGINT/SIGTERM triggers graceful shutdown: closes socket, waits for ffmpeg to finalize MP4

## Key Properties

| Property | Value |
|----------|-------|
| Depends | `selkies` (capture bridge + stream), `ffmpeg` (muxing) |
| Install | `~/.local/bin/pixelflux-record` (Python script) |
| Capture | Via `/tmp/ov-capture.sock` (selkies WebSocket bridge) |
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

## Architecture

```
selkies process (existing capture, unchanged)
    ├── ScreenCapture (captures full composited desktop)
    ├── WebSocket server :8081 (broadcasts H.264 frames)
    └── Capture bridge thread (internal WebSocket client)
        └── Unix socket /tmp/ov-capture.sock
            └── pixelflux-record connects here (STREAM mode)
                └── pipes H.264 frames to ffmpeg
```

## Included In

- `selkies-desktop` metalayer

## Cross-References

- `/ov:record` -- `ov record start --mode desktop` auto-detects pixelflux-record
- `/ov-layers:wl-screenshot-pixelflux` -- Screenshot companion (same capture bridge)
- `/ov-layers:wf-recorder` -- Alternative for sway-desktop (wlr-screencopy)
- `/ov-layers:selkies` -- Parent layer (provides capture bridge + WebSocket stream)
- `/ov-layers:ffmpeg` -- Required dependency (MP4 muxing)

## When to Use This Skill

Use when the user asks about:
- Desktop video recording on selkies-desktop
- The pixelflux recording pipeline
- The `wl-record-pixelflux` layer
