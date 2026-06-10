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
charly eval wl exec <image> xterm           # Launch xterm (triggers XWayland)
charly eval wl focus <image> xterm          # Focus by app_id
charly eval wl xprop <image> xterm          # Query X11 properties
charly eval wl geometry <image> xterm       # Get window position/size
charly eval wl close <image> xterm          # Close window
```

**Important:** The `charly eval wl exec` command sets `DISPLAY=:0` automatically for X11 apps.

## Why It's in selkies-desktop

labwc has XWayland compiled in (`+xwayland`) but starts it on-demand only when an X11 client opens. Without an X11 app like xterm:
- XWayland never starts
- xdotool, xprop, xwininfo find no windows
- `charly eval wl scroll` (xdotool click 4/5) and `charly eval wl drag` (xdotool mousedown/mouseup) can't target X11 windows

With xterm installed, users can launch it to enable full XWayland testing.

## Included In

- `selkies-desktop` metalayer

## Used In Images

- `/charly-selkies:selkies-labwc` (via `selkies-desktop` metalayer)
- `/charly-selkies:selkies-labwc-nvidia` (via `selkies-desktop` metalayer)

## Cross-References

- `/charly-eval:wl` — `charly eval wl exec`, `charly eval wl focus`, `charly eval wl close`, `charly eval wl xprop`, `charly eval wl geometry`
- `/charly-selkies:selkies-desktop-layer` — Desktop metalayer that includes this layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
