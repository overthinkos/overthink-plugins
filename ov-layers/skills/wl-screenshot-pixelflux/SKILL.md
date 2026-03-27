# wl-screenshot-pixelflux - Screenshot via pixelflux rendering pipeline

## Overview

Provides `pixelflux-screenshot` for capturing screenshots on selkies-desktop. Taps directly into pixelflux's `ScreenCapture` API — the same rendering pipeline that powers the WebSocket video stream. Zero-copy, no protocol limitations.

**Why not grim?** labwc running nested inside pixelflux advertises `wlr-screencopy` but can't deliver frames because pixelflux owns the render target. `pixelflux-screenshot` bypasses this by capturing from the rendering pipeline directly.

## Layer Definition

```yaml
depends:
  - selkies
```

No RPM packages — uses pixelflux and Pillow from the selkies layer's pixi environment.

## How It Works

1. Creates a `ScreenCapture` instance with `output_mode=0` (JPEG), quality 95
2. Callback receives JPEG stripes (11 per frame at 720p, 16 per frame at 1080p)
3. Each stripe has a 4-byte header (frame_id + y_offset) then standard JPEG data
4. Strips headers, decodes each JPEG with Pillow, stitches vertically
5. Outputs PNG to stdout

## Key Properties

| Property | Value |
|----------|-------|
| Depends | `selkies` (pixelflux + Pillow in pixi env) |
| Install | `~/.local/bin/pixelflux-screenshot` (Python script) |
| Protocol | pixelflux `ScreenCapture` API (not Wayland screencopy) |

## Usage

Used by `ov wl screenshot` — auto-detected when `pixelflux-screenshot` is available (preferred over grim).

```bash
ov wl screenshot <image> [output.png]
```

## Included In

- `selkies-desktop` metalayer

## Cross-references

- `/ov:wl` — `ov wl screenshot` auto-detects pixelflux-screenshot
- `/ov-layers:wl-screenshot-grim` — Alternative for sway-desktop
- `/ov-layers:selkies` — Parent layer (provides pixelflux + Pillow)
