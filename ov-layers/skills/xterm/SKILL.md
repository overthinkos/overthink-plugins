# xterm - X11 Terminal (XWayland)

## Overview

Lightweight X11 terminal emulator. On labwc (selkies-desktop), launching xterm triggers XWayland to start on-demand, enabling X11-based automation tools (xdotool, xprop, xwininfo) to find windows.

## Layer Definition

```yaml
rpm:
  packages:
    - xterm
```

## Key Properties

| Property | Value |
|----------|-------|
| Depends | None |
| Packages | `xterm` |
| WM_CLASS | `xterm` / `XTerm` |
| XWayland | Triggers on-demand start on labwc |

## Usage

```bash
ov wl exec <image> xterm           # Launch xterm (triggers XWayland)
ov wl focus <image> xterm          # Focus by app_id
ov wl xprop <image> xterm          # Query X11 properties
ov wl geometry <image> xterm       # Get window position/size
ov wl close <image> xterm          # Close window
```

**Important:** The `ov wl exec` command sets `DISPLAY=:0` automatically for X11 apps.

## Why It's in selkies-desktop

labwc has XWayland compiled in (`+xwayland`) but starts it on-demand only when an X11 client opens. Without an X11 app like xterm:
- XWayland never starts
- xdotool, xprop, xwininfo find no windows
- `ov wl scroll` (xdotool click 4/5) and `ov wl drag` (xdotool mousedown/mouseup) can't target X11 windows

With xterm installed, users can launch it to enable full XWayland testing.

## Included In

- `selkies-desktop` metalayer

## Used In Images

- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Cross-References

- `/ov:wl` — `ov wl exec`, `ov wl focus`, `ov wl close`, `ov wl xprop`, `ov wl geometry`
- `/ov-layers:selkies-desktop` — Desktop metalayer that includes this layer
