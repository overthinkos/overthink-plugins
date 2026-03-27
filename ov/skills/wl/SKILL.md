# WL - Compositor-Agnostic Desktop Automation

## Overview

`ov wl` is the unified desktop automation command for all wlroots-based compositors (sway, labwc, niri). It provides screenshots, input (click, type, key combos, scroll, drag), window management (via `wlrctl toplevel`), clipboard, resolution control, accessibility introspection (AT-SPI2), and window geometry queries. Works on both sway-desktop and selkies-desktop images.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Screenshot | `ov wl screenshot <image> [file]` | Capture desktop as PNG via grim |
| Click | `ov wl click <image> <x> <y>` | Click at absolute coordinates via wlrctl |
| Double-click | `ov wl double-click <image> <x> <y>` | Double-click with configurable delay |
| Type text | `ov wl type <image> <text>` | Send keyboard input via wtype |
| Send key | `ov wl key <image> <key-name>` | Press a named key via wtype |
| Key combo | `ov wl key-combo <image> <keys>` | Send key combination (ctrl+c, alt+tab) |
| Move mouse | `ov wl mouse <image> <x> <y>` | Move pointer to absolute coordinates |
| Scroll | `ov wl scroll <image> <x> <y> <dir>` | Scroll at coordinates (up/down/left/right) |
| Drag | `ov wl drag <image> <x1> <y1> <x2> <y2>` | Drag between coordinates (experimental) |
| List windows | `ov wl windows <image>` | List windows (wlrctl toplevel, xdotool fallback) |
| List toplevel | `ov wl toplevel <image>` | List Wayland toplevel windows via wlrctl |
| Focus window | `ov wl focus <image> <title>` | Focus window (wlrctl toplevel, xdotool fallback) |
| Close window | `ov wl close <image> <title>` | Close window via wlrctl toplevel |
| Fullscreen | `ov wl fullscreen <image> <title>` | Toggle fullscreen via wlrctl toplevel |
| Minimize | `ov wl minimize <image> <title>` | Toggle minimize via wlrctl toplevel |
| Launch app | `ov wl exec <image> <command>` | Launch application in container |
| Resolution | `ov wl resolution <image> <WxH>` | Set output resolution via wlr-randr |
| Clipboard | `ov wl clipboard <image> get/set/clear` | Read/write Wayland clipboard |
| Window props | `ov wl xprop <image> [target]` | Query X11 window properties |
| Window rect | `ov wl geometry <image> <target>` | Get window position/size as JSON |
| A11y tree | `ov wl atspi <image> tree` | Dump accessibility tree as JSON |
| A11y find | `ov wl atspi <image> find <query>` | Find elements by name/role |
| A11y click | `ov wl atspi <image> click <query>` | Click element by name/role |
| Status | `ov wl status <image>` | Check all tool availability |

## Compositor Compatibility

All commands work on any wlroots-based compositor:

| Tool | Protocol | sway | labwc (selkies) | niri |
|------|----------|------|-----------------|------|
| grim | wlr-screencopy | YES | NO (nested compositor) | YES |
| pixelflux-screenshot | pixelflux API | NO | YES | NO |
| wtype | zwp_virtual_keyboard_v1 | YES | YES | YES |
| wlrctl pointer | wlr-virtual-pointer | YES | YES | YES |
| wlrctl toplevel | wlr-foreign-toplevel-management | YES | YES | YES |
| wlr-randr | wlr-output-management | YES | YES | YES |
| wl-copy/paste | wlr-data-control | YES | YES | YES |
| xdotool | X11 (XWayland) | YES | YES (on-demand) | YES |
| swaymsg | i3 IPC | YES | NO | NO |

**Coordinate translation flags:**
- `--from-cdp` — Works on all compositors (uses `window.screenX/Y`)
- `--from-sway` — Sway only (uses `swaymsg -t get_tree`)
- `--from-x11` — Works on all compositors with XWayland

## Architecture

```
CLI command -> resolveContainer (engine + container name)
           -> exec sh -c "export WAYLAND_DISPLAY=... && <tool command>"
           -> capture stdout (screenshot) or run silently (input)
```

Uses `exec` into the container. All tools use native Wayland protocols — no daemon, no `/dev/uinput`, no VNC server required.

## Requirements

- Container must include `wl-tools` layer (wtype, wlrctl, wl-clipboard, wlr-randr, xdotool, ydotool)
- For screenshots: `wl-screenshot-grim` (sway) or `wl-screenshot-pixelflux` (selkies)
- Container must have a running Wayland compositor (sway, labwc, etc.)
- For AT-SPI2: `a11y-tools` layer (python3-pyatspi, python3-gobject) + `dbus` layer
- For XWayland: an X11 app like `xterm` must be running to trigger XWayland start on labwc
- Included in `sway-desktop` and `selkies-desktop` metalayers

## Commands

### Key Combo
```bash
ov wl key-combo my-app ctrl+c          # Ctrl+C
ov wl key-combo my-app alt+tab         # Alt+Tab
ov wl key-combo my-app ctrl+shift+t    # Ctrl+Shift+T
ov wl key-combo my-app super+l         # Super+L (lock)
```

Modifiers: `ctrl`/`control`, `alt`, `shift`, `super`/`win`/`logo`, `meta`. Uses `wtype -M`.

### Scroll
```bash
ov wl scroll my-app 960 540 down              # scroll down 3 steps at center
ov wl scroll my-app 960 540 up --amount 10    # scroll up 10 steps
```

Uses xdotool click 4/5/6/7 (X11 scroll buttons) for XWayland windows. Falls back to wtype Page_Up/Page_Down.

### Drag (Experimental)
```bash
ov wl drag my-app 100 100 500 500                # drag left to right
ov wl drag my-app 100 100 500 500 --duration 500  # slower drag (500ms)
```

Requires XWayland (uses `xdotool mousemove + mousedown/mouseup`).

### Window Management (wlrctl toplevel)
```bash
ov wl toplevel my-app                  # list all windows
ov wl focus my-app "Chrome"            # focus by title
ov wl close my-app "Chrome"            # close by title
ov wl fullscreen my-app "Chrome"       # toggle fullscreen
ov wl minimize my-app "Chrome"         # toggle minimize
ov wl exec my-app foot                 # launch terminal
```

### Resolution
```bash
ov wl resolution my-app 1920x1080              # auto-detect output
ov wl resolution my-app 2560x1440 -o WL-1      # specific output
```

### Clipboard
```bash
ov wl clipboard my-app get                   # read clipboard
ov wl clipboard my-app set "hello"           # write clipboard
ov wl clipboard my-app clear                 # clear clipboard
ov wl clipboard my-app get --primary         # read primary selection
```

### Window Geometry
```bash
ov wl geometry my-app "Chrome"    # returns JSON: {"x":0,"y":0,"width":1920,"height":1080}
ov wl xprop my-app                # active window properties
ov wl xprop my-app "Chrome"      # specific window properties
```

### AT-SPI2 Accessibility
```bash
ov wl atspi my-app tree                    # dump full accessibility tree
ov wl atspi my-app find "Save"             # find elements named "Save"
ov wl atspi my-app find "button"           # find elements with role "button"
ov wl atspi my-app find "Save:button"      # find by name AND role
ov wl atspi my-app click "Save:button"     # click element by name/role
```

Requires `a11y-tools` layer. Chrome needs `--force-renderer-accessibility` flag.

### CDP → WL Bridge

Use `ov cdp click --wl` to find elements by CSS selector in Chrome and deliver clicks via wlrctl (critical for selkies-desktop which has no VNC):

```bash
ov cdp click selkies-desktop $TAB '#submit-button' --wl
```

## Differences from VNC and Sway Commands

| Aspect | `ov wl` | `ov vnc` | `ov sway` |
|--------|---------|----------|-----------|
| Compositors | All wlroots | Requires wayvnc | Sway only |
| Transport | exec into container | TCP port 5900 | exec into container |
| Window mgmt | wlrctl toplevel | No | swaymsg IPC |
| Clipboard | wl-copy/paste | rfb cut-text | No |
| Remote access | No | Yes (TCP) | No |
| NVIDIA headless | Works | Gray screen bug | Works |

Source: `ov/wl.go`.

## Cross-references

- `/ov:vnc` — VNC/RFB protocol alternative (TCP-based, works remotely)
- `/ov:cdp` — Chrome DevTools Protocol (DOM-level interaction, `--wl` flag for click, `axtree` for accessibility)
- `/ov:sway` — Sway-specific compositor control (tree, workspaces, layout, move, resize)
- `/ov-layers:wl-tools` — Compositor-agnostic tools (wtype, wlrctl, wl-clipboard, wlr-randr, xdotool, ydotool)
- `/ov-layers:wl-screenshot-grim` — Screenshot layer for sway (grim, wlr-screencopy)
- `/ov-layers:wl-screenshot-pixelflux` — Screenshot layer for selkies (pixelflux rendering pipeline)
- `/ov-layers:a11y-tools` — AT-SPI2 accessibility (python3-pyatspi, python3-gobject)
- `/ov-layers:xterm` — X11 terminal for XWayland testing
- `/ov-layers:sway-desktop` — Desktop metalayer (wl-tools + wl-screenshot-grim)
- `/ov-layers:selkies-desktop` — Desktop metalayer (wl-tools + wl-screenshot-pixelflux + a11y-tools + xterm)
