---
name: wl-overlay
description: |
  Fullscreen Wayland overlays for screen recordings via gtk4-layer-shell.
  MUST be invoked before any work involving: ov eval wl overlay commands, recording
  overlays, title cards, lower-thirds, countdowns, or fade transitions.
---

# WL Overlay -- Wayland Desktop Overlays for Recording

## Overview

`ov eval wl overlay` creates and manages fullscreen Wayland overlays using the layer-shell protocol via gtk4-layer-shell. Overlays render directly in the compositor, appearing natively in screen recordings without post-production editing. Supports true alpha transparency — desktop content is visible through semi-transparent overlays.

## Architecture

Three components work together:

1. **`wl-overlay` layer** — installs `gtk4-layer-shell`, `gtk4`, `python3-gobject` RPMs + `ov-overlay` Python daemon/client
2. **`ov eval wl overlay` Go commands** — CLI that resolves the container and invokes `ov-overlay` inside it
3. **`ov-overlay` daemon** — GTK4 application managing multiple named overlay windows via Unix socket IPC

The daemon starts on-demand in a tmux session (`ov-overlay-daemon`) when the first `show` command is issued. It persists until `hide --all` or the tmux session is killed.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Show overlay | `ov eval wl overlay show <image> --type text --text "Hello"` | Display a named overlay |
| Hide one | `ov eval wl overlay hide <image> --name intro` | Remove specific overlay |
| Hide all | `ov eval wl overlay hide <image> --all` | Remove all overlays |
| List | `ov eval wl overlay list <image>` | List active overlays (JSON) |
| Status | `ov eval wl overlay status <image>` | Check daemon health |

## Overlay Types

### Text (Title Card)

Full-screen semi-transparent overlay with centered text. Use for intro/outro title cards.

```bash
ov eval wl overlay show myimage --type text \
  --text "Building a REST API" \
  --bg "rgba(0,0,0,0.7)" \
  --font-size 64 --name title
```

### Lower-Third

Bottom-anchored bar with name and optional subtitle. Use for speaker identification.

```bash
ov eval wl overlay show myimage --type lower-third \
  --text "Andreas Trawoeger" \
  --subtitle "Developer" --name speaker
```

### Watermark

Corner-anchored persistent text at low opacity. Use for draft/preview markers.

```bash
ov eval wl overlay show myimage --type watermark \
  --text "DRAFT" --position bottom-right \
  --color red --opacity 0.5 --name wm
```

Default opacity is 0.3. Positions: `top-left`, `top-right`, `bottom-left`, `bottom-right`, `top`, `bottom`.

### Countdown

Full-screen animated countdown that auto-hides on completion. Use before recording starts.

```bash
ov eval wl overlay show myimage --type countdown --seconds 5 --name cd
```

### Highlight

Transparent overlay with a colored rectangle at specified coordinates. Use to draw attention to a screen region.

```bash
ov eval wl overlay show myimage --type highlight \
  --region "430,290,510,50" \
  --color "rgba(255,0,0,0.4)" --name hl
```

Region format: `X,Y,Width,Height` in pixels.

### Fade

Full-screen solid color overlay. Use for transitions (fade to black, fade to white).

```bash
ov eval wl overlay show myimage --type fade --color black --name outro
```

## Duration Auto-Hide

Any overlay (except countdown, which has built-in auto-hide) can auto-remove after a duration:

```bash
ov eval wl overlay show myimage --type text --text "INTRO" --duration 5s --name intro
```

Supported formats: `5s`, `1.5m`, `500ms`, or bare seconds (`5`).

## Recording Workflow

Overlays compose naturally with `ov eval record`:

```bash
# Title card
ov eval wl overlay show myimage --type text --text "Building a REST API" \
  --bg "rgba(0,0,0,0.9)" --font-size 64 --name title

# Start recording
ov eval record start myimage -n demo -m desktop

# Fade out title after 3s
sleep 3
ov eval wl overlay hide myimage --name title

# Lower-third speaker ID
ov eval wl overlay show myimage --type lower-third \
  --text "Andreas Trawoeger" --subtitle "Developer" --name speaker

# ... record content ...

# Fade to black for ending
ov eval wl overlay show myimage --type fade --color black --name outro
sleep 2

# Stop recording
ov eval record stop myimage -n demo -o demo.mp4
ov eval wl overlay hide myimage --all
```

## Compositor Compatibility

All overlay types work on all wlroots-based compositors:

| Compositor | Environment | Status | Notes |
|-----------|-------------|--------|-------|
| sway | sway-browser-vnc | WORKS | Instant rendering, captured by wayvnc |
| labwc | selkies-desktop | WORKS | ~15s capture latency in controller mode (no browser). Instant in recordings |
| niri | niri-desktop | WORKS | Layer-shell supported via Smithay |

### Key Technical Details

- **True RGBA transparency**: enabled by `window { background: none; }` CSS at daemon startup. Without this, GTK4 on Wayland creates opaque surfaces
- **Cairo-drawn backgrounds**: backgrounds are painted via `Gtk.DrawingArea` + Cairo, not CSS. GTK4 CSS `background-color` with `rgba()` does not composite correctly on layer-shell windows
- **Layer**: uses `Gtk4LayerShell.Layer.OVERLAY` (highest z-order, above waybar/swaync)
- **Keyboard**: `KeyboardMode.NONE` — overlays never capture input
- **Exclusive zone**: `-1` — overlays don't push other windows
- **Python interpreter**: uses `/usr/bin/python3` (system Python with PyGObject), not pixi's Python

### Selkies-Desktop Capture Latency

On selkies-desktop, `pixelflux-screenshot` captures from the H.264 WebSocket stream. In controller mode (no browser client connected), the frame rate is very low — screenshots may take 15-30 seconds to reflect overlay changes. **This does NOT affect recordings** — `ov eval record` captures at 30fps, so overlays appear immediately in recorded video.

## IPC Protocol

Socket: `/tmp/ov-overlay.sock` (Unix domain, JSON-line)

```json
{"action":"show","type":"text","text":"Hello","name":"intro","bg":"rgba(0,0,0,0.7)"}
{"action":"hide","name":"intro"}
{"action":"hide_all"}
{"action":"list"}
{"action":"status"}
```

## Show Command Flags

| Flag | Default | Description |
|------|---------|-------------|
| `--type` | (required) | `text`, `lower-third`, `watermark`, `countdown`, `highlight`, `fade` |
| `--text` | `""` | Text content |
| `--subtitle` | `""` | Subtitle (lower-third only) |
| `--name` | auto-generated | Overlay identifier for hide/list |
| `--position` | `center` | Positioning (watermark: corner anchoring) |
| `--bg` | type-dependent | Background color (CSS rgba or named) |
| `--color` | `white` | Text/highlight color |
| `--font-size` | `48` | Font size in pixels |
| `--opacity` | `1.0` (watermark: `0.3`) | Overall window opacity |
| `--duration` | none | Auto-hide after duration (e.g. `5s`) |
| `--seconds` | `3` | Countdown seconds |
| `--region` | `100,100,400,300` | Highlight region `X,Y,W,H` |

## Requirements

- Container must include `wl-overlay` layer (gtk4-layer-shell + python3-gobject)
- Container must have a running Wayland compositor (sway, labwc, niri)
- Container must have `tmux` layer (for daemon hosting)
- Included in `sway-desktop`, `selkies-desktop`, and `niri-desktop` metalayers

## Cross-References

- `/ov:wl` — Compositor-agnostic desktop automation (screenshots, input, windows)
- `/ov:record` — Terminal and desktop recording (overlays compose with recording workflow)
- `/ov:cdp` — Chrome DevTools Protocol (DOM-level interaction)
- `/ov-layers:wl-overlay` — Layer reference (RPM packages, dependencies)
- `/ov-layers:sway-desktop` — Desktop metalayer (includes wl-overlay)
- `/ov-layers:selkies-desktop` — Desktop metalayer (includes wl-overlay)
- `/ov-layers:niri-desktop` — Desktop metalayer (includes wl-overlay)

Source: `ov/wl_overlay.go` (Go commands), `layers/wl-overlay/ov-overlay` (Python daemon/client).
