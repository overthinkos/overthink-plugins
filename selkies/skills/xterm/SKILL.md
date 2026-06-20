---
name: xterm
description: |
  Lightweight X11 terminal emulator (xterm) that triggers on-demand XWayland on labwc/selkies-desktop, enabling X11 automation tools (xdotool, xprop, xwininfo) to find windows.
  Use when working with xterm, on-demand XWayland startup, or X11-based window automation on a Wayland desktop.
---

# xterm - X11 Terminal (XWayland)

## Overview

Lightweight X11 terminal emulator. On labwc (selkies-desktop), launching xterm triggers XWayland to start on-demand, enabling X11-based automation tools (xdotool, xprop, xwininfo) to find windows.

## Candy Definition

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
charly check wl exec <image> xterm           # Launch xterm (triggers XWayland)
charly check wl focus <image> xterm          # Focus by app_id
charly check wl xprop <image> xterm          # Query X11 properties
charly check wl geometry <image> xterm       # Get window position/size
charly check wl close <image> xterm          # Close window
```

**Important:** The `charly check wl exec` command sets `DISPLAY=:0` automatically for X11 apps.

## Why It's in selkies-desktop

labwc has XWayland compiled in (`+xwayland`) but starts it on-demand only when an X11 client opens. Without an X11 app like xterm:
- XWayland never starts
- xdotool, xprop, xwininfo find no windows
- `charly check wl scroll` (xdotool click 4/5) and `charly check wl drag` (xdotool mousedown/mouseup) can't target X11 windows

With xterm installed, users can launch it to enable full XWayland testing.

## Included In

- `selkies-desktop` metalayer

## Used In Boxes

- `/charly-selkies:selkies-labwc` (via `selkies-desktop` metalayer)
- `/charly-selkies:selkies-labwc-nvidia` (via `selkies-desktop` metalayer)

## Cross-References

- `/charly-check:wl` — `charly check wl exec`, `charly check wl focus`, `charly check wl close`, `charly check wl xprop`, `charly check wl geometry`
- `/charly-selkies:selkies-desktop-layer` — Desktop metalayer that includes this candy

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-check:check` — declarative testing (`check:` block, `charly check box`, `charly check live`)
