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
ov test wl exec <image> xterm           # Launch xterm (triggers XWayland)
ov test wl focus <image> xterm          # Focus by app_id
ov test wl xprop <image> xterm          # Query X11 properties
ov test wl geometry <image> xterm       # Get window position/size
ov test wl close <image> xterm          # Close window
```

**Important:** The `ov test wl exec` command sets `DISPLAY=:0` automatically for X11 apps.

## Why It's in selkies-desktop

labwc has XWayland compiled in (`+xwayland`) but starts it on-demand only when an X11 client opens. Without an X11 app like xterm:
- XWayland never starts
- xdotool, xprop, xwininfo find no windows
- `ov test wl scroll` (xdotool click 4/5) and `ov test wl drag` (xdotool mousedown/mouseup) can't target X11 windows

With xterm installed, users can launch it to enable full XWayland testing.

## Included In

- `selkies-desktop` metalayer

## Used In Images

- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Cross-References

- `/ov:wl` — `ov test wl exec`, `ov test wl focus`, `ov test wl close`, `ov test wl xprop`, `ov test wl geometry`
- `/ov-layers:selkies-desktop` — Desktop metalayer that includes this layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
