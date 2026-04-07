---
name: wl-screenshot-pixelflux
description: |
  Screenshot via selkies WebSocket capture bridge for selkies-desktop.
  Use when working with the wl-screenshot-pixelflux layer.
---

# wl-screenshot-pixelflux -- Screenshot via selkies capture bridge

## Overview

Provides `pixelflux-screenshot` for capturing screenshots on selkies-desktop. Connects to the in-process capture bridge at `/tmp/ov-capture.sock` (started by `selkies-capture-server` inside the selkies process). The capture bridge taps into the selkies WebSocket stream and decodes H.264 frames to PNG via ffmpeg.

**Why not grim?** labwc running nested inside pixelflux can't deliver wlr-screencopy frames. The capture bridge bypasses this by tapping into the selkies WebSocket stream which has direct access to the composited desktop.

## Layer Definition

```yaml
depends:
  - selkies
```

No RPM packages — uses the capture bridge provided by the selkies layer.

## How It Works

1. Connects to Unix socket at `/tmp/ov-capture.sock`
2. Sends `SCREENSHOT\n`
3. Receives `4-byte length + PNG data` response (or `0 + reason string` on failure)
4. Validates PNG completeness (reports incomplete reads)
5. Outputs PNG to stdout

The capture bridge (running inside the selkies process) handles frame collection, H.264 NAL filtering (drops Opus audio), and ffmpeg decode. On failure, the bridge sends a descriptive reason (e.g., "not connected", "ffmpeg exit N: ...") instead of bare `exit status 1`.

## Status Query

```bash
pixelflux-screenshot --status
# Returns: {"connected": true, "mode": "controller", "frames": 90, "seq": 1234, "active_streams": 0, "last_error": ""}
```

Fields: `connected` (WebSocket up), `mode` (controller/viewer/reconnecting), `frames` (buffered H.264), `seq` (total received), `active_streams` (active recordings), `last_error` (last ffmpeg error).

## Key Properties

| Property | Value |
|----------|-------|
| Depends | `selkies` (capture bridge in selkies process) |
| Install | `~/.local/bin/pixelflux-screenshot` (Python script) |
| Capture | Via `/tmp/ov-capture.sock` (selkies WebSocket bridge) |

## Usage

Used by `ov wl screenshot` — auto-detected when `pixelflux-screenshot` is available (preferred over grim).

```bash
ov wl screenshot <image> [output.png]
```

## Architecture

```
selkies process
    ├── WebSocket :8081 (H.264 frame broadcast)
    └── Capture bridge thread → /tmp/ov-capture.sock
        └── SCREENSHOT request → ffmpeg H.264→PNG decode → PNG response
```

## Included In

- `selkies-desktop` metalayer

## Used In Images

- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Cross-References

- `/ov:wl` — `ov wl screenshot` auto-detects pixelflux-screenshot
- `/ov-layers:wl-record-pixelflux` — Recording companion (same capture bridge)
- `/ov-layers:wl-screenshot-grim` — Alternative for sway-desktop (wlr-screencopy)
- `/ov-layers:selkies` — Parent layer (provides capture bridge)
