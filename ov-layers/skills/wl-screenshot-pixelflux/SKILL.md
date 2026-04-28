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

Used by `ov eval wl screenshot` — auto-detected when `pixelflux-screenshot` is available (preferred over grim).

```bash
ov eval wl screenshot <image> [output.png]
```

## Architecture

```
selkies process (single ScreenCapture singleton)
    ├── WebSocket :8081 (H.264 frame broadcast)
    └── Capture bridge thread → /tmp/ov-capture.sock
        └── SCREENSHOT request → ffmpeg H.264→PNG decode → PNG response
```

**Singleton guarantee:** The `ScreenCapture` instance inside selkies is process-wide. `pixelflux-screenshot` taps into the **same** capture path that the browser client uses and that `pixelflux-record` (`/ov-layers:wl-record-pixelflux`) uses — there is never a second capture process spawned. This matters because pixelflux's `WaylandBackend` is expensive to construct (creates EGL context, dmabuf allocators, GPU texture pools), so spawning a new one per screenshot would leak GBM buffers on every call. The singleton was re-affirmed in commit `6be85eb` (`selkies ScreenCapture singleton to stop pixelflux WaylandBackend leak`) and paired with the per-frame `cleanup_texture_cache()` fix in commit `7977b91`. See `/ov-layers:selkies` (Pixelflux Memory Management) for the full leak diagnosis and fix.

## Included In

- `selkies-desktop` metalayer

## Used In Images

- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Cross-References

- `/ov:wl` — `ov eval wl screenshot` auto-detects pixelflux-screenshot
- `/ov-layers:wl-record-pixelflux` — Recording companion (same capture bridge + same singleton)
- `/ov-layers:wl-screenshot-grim` — Alternative for sway-desktop (wlr-screencopy)
- `/ov-layers:selkies` — Parent layer providing the ScreenCapture singleton and capture bridge
- `/ov-layers:selkies-desktop` — Metalayer that composes this screenshot path into the selkies-desktop image
- `/ov:record` — Uses `/ov-layers:wl-record-pixelflux` via the same singleton

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
