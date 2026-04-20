---
name: wl
description: |
  MUST be invoked before any work involving: Wayland / wlroots desktop automation — `ov test wl` commands (screenshots, click/type/scroll/drag, window management via wlrctl, clipboard, resolution control, AT-SPI2 introspection, window geometry), nested `wl sway` / `wl overlay` subcommands, or `wl:` declarative verbs inside `tests:` blocks. Covers sway-desktop and selkies-desktop image automation on both sway and labwc compositors.
---

# WL - Compositor-Agnostic Desktop Automation

## Overview

`ov test wl` is the unified desktop automation command for all wlroots-based compositors (sway, labwc, niri). It provides screenshots, input (click, type, key combos, scroll, drag), window management (via `wlrctl toplevel`), clipboard, resolution control, accessibility introspection (AT-SPI2), and window geometry queries. Works on both sway-desktop and selkies-desktop images.

### Also as a declarative verb

Every `ov test wl <method>` (including nested `wl overlay <method>` and `wl sway <method>`) is authorable as a `wl:` verb inside a `tests:` block. Nested subcommands are hyphenated in YAML: `wl: overlay-show`, `wl: sway-tree`, `wl: sway-workspaces`. Method-specific fields (`x`, `y`, `text`, `key`, `combo`, `target`, `action`, `artifact`) are siblings of the verb line. See `/ov:test` for the full method allowlist. Example: `- wl: screenshot\n  artifact: /tmp/desktop.png\n  artifact_min_bytes: 10000`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Screenshot | `ov test wl screenshot <image> [file]` | Capture desktop as PNG via grim |
| Click | `ov test wl click <image> <x> <y>` | Click at absolute coordinates via wlrctl |
| Double-click | `ov test wl double-click <image> <x> <y>` | Double-click with configurable delay |
| Type text | `ov test wl type <image> <text>` | Send keyboard input via wtype |
| Send key | `ov test wl key <image> <key-name>` | Press a named key via wtype |
| Key combo | `ov test wl key-combo <image> <keys>` | Send key combination (ctrl+c, alt+tab) |
| Move mouse | `ov test wl mouse <image> <x> <y>` | Move pointer to absolute coordinates |
| Scroll | `ov test wl scroll <image> <x> <y> <dir>` | Scroll at coordinates (up/down/left/right) |
| Drag | `ov test wl drag <image> <x1> <y1> <x2> <y2>` | Drag between coordinates (experimental) |
| List windows | `ov test wl windows <image>` | List windows (wlrctl toplevel, xdotool fallback) |
| List toplevel | `ov test wl toplevel <image>` | List Wayland toplevel windows via wlrctl |
| Focus window | `ov test wl focus <image> <title>` | Focus window (wlrctl toplevel, xdotool fallback) |
| Close window | `ov test wl close <image> <title>` | Close window via wlrctl toplevel |
| Fullscreen | `ov test wl fullscreen <image> <title>` | Toggle fullscreen via wlrctl toplevel |
| Minimize | `ov test wl minimize <image> <title>` | Toggle minimize via wlrctl toplevel |
| Launch app | `ov test wl exec <image> <command>` | Launch application in container |
| Resolution | `ov test wl resolution <image> <WxH>` | Set output resolution via wlr-randr |
| Clipboard | `ov test wl clipboard <image> get/set/clear` | Read/write Wayland clipboard |
| Window props | `ov test wl xprop <image> [target]` | Query X11 window properties |
| Window rect | `ov test wl geometry <image> <target>` | Get window position/size as JSON |
| A11y tree | `ov test wl atspi <image> tree` | Dump accessibility tree as JSON |
| A11y find | `ov test wl atspi <image> find <query>` | Find elements by name/role |
| A11y click | `ov test wl atspi <image> click <query>` | Click element by name/role |
| Status | `ov test wl status <image>` | Check all tool availability |
| Overlay show | `ov test wl overlay show <image> --type text --text "Hello"` | Show recording overlay (see `/ov:wl-overlay`) |
| Overlay hide | `ov test wl overlay hide <image> --all` | Remove all overlays |

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
ov test wl key-combo my-app ctrl+c          # Ctrl+C
ov test wl key-combo my-app alt+tab         # Alt+Tab
ov test wl key-combo my-app ctrl+shift+t    # Ctrl+Shift+T
ov test wl key-combo my-app super+l         # Super+L (lock)
```

Modifiers: `ctrl`/`control`, `alt`, `shift`, `super`/`win`/`logo`, `meta`. Uses `wtype -M`.

### Scroll
```bash
ov test wl scroll my-app 960 540 down              # scroll down 3 steps at center
ov test wl scroll my-app 960 540 up --amount 10    # scroll up 10 steps
```

Uses xdotool click 4/5/6/7 (X11 scroll buttons) for XWayland windows. Falls back to wtype Page_Up/Page_Down.

### Drag (Experimental)
```bash
ov test wl drag my-app 100 100 500 500                # drag left to right
ov test wl drag my-app 100 100 500 500 --duration 500  # slower drag (500ms)
```

Requires XWayland (uses `xdotool mousemove + mousedown/mouseup`).

### Window Management (wlrctl toplevel)
```bash
ov test wl toplevel my-app                  # list all windows
ov test wl focus my-app "Chrome"            # focus by title
ov test wl close my-app "Chrome"            # close by title
ov test wl fullscreen my-app "Chrome"       # toggle fullscreen
ov test wl minimize my-app "Chrome"         # toggle minimize
ov test wl exec my-app foot                 # launch terminal
```

### Resolution
```bash
ov test wl resolution my-app 1920x1080              # auto-detect output
ov test wl resolution my-app 2560x1440 -o WL-1      # specific output
```

### Clipboard
```bash
ov test wl clipboard my-app get                   # read clipboard
ov test wl clipboard my-app set "hello"           # write clipboard
ov test wl clipboard my-app clear                 # clear clipboard
ov test wl clipboard my-app get --primary         # read primary selection
```

### Window Geometry
```bash
ov test wl geometry my-app "Chrome"    # returns JSON: {"x":0,"y":0,"width":1920,"height":1080}
ov test wl xprop my-app                # active window properties
ov test wl xprop my-app "Chrome"      # specific window properties
```

### AT-SPI2 Accessibility
```bash
ov test wl atspi my-app tree                    # dump full accessibility tree
ov test wl atspi my-app find "Save"             # find elements named "Save"
ov test wl atspi my-app find "button"           # find elements with role "button"
ov test wl atspi my-app find "Save:button"      # find by name AND role
ov test wl atspi my-app click "Save:button"     # click element by name/role
```

Requires `a11y-tools` layer. Chrome needs `--force-renderer-accessibility` flag.

### CDP → WL Bridge

Use `ov test cdp click --wl` to find elements by CSS selector in Chrome and deliver clicks via wlrctl (critical for selkies-desktop which has no VNC):

```bash
ov test cdp click selkies-desktop $TAB '#submit-button' --wl
```

## Sway-Specific Commands (`ov test wl sway`)

Sway IPC commands are grouped under `ov test wl sway <subcommand>`. These require a sway compositor and use swaymsg. They will error on labwc/niri.

```bash
ov test wl sway tree <image>              # Get sway window tree (JSON)
ov test wl sway msg <image> 'focus left'  # Run any swaymsg command
ov test wl sway workspaces <image>        # List workspaces (JSON)
ov test wl sway outputs <image>           # List outputs (JSON)
ov test wl sway focus <image> left        # Focus by direction or [criteria]
ov test wl sway move <image> right        # Move focused window
ov test wl sway resize <image> width 10px # Resize focused window
ov test wl sway kill <image>              # Close focused window
ov test wl sway floating <image>          # Toggle floating
ov test wl sway layout <image> tabbed     # Set layout mode
ov test wl sway workspace <image> 2       # Switch workspace
ov test wl sway reload <image>            # Reload sway config
```

## Differences from VNC

| Aspect | `ov test wl` | `ov test vnc` |
|--------|---------|----------|
| Compositors | All wlroots (+ sway subgroup) | Requires wayvnc |
| Transport | exec into container | TCP port 5900 |
| Window mgmt | wlrctl toplevel + sway IPC | No |
| Clipboard | wl-copy/paste | rfb cut-text |
| Remote access | No | Yes (TCP) |
| NVIDIA headless | Works | Works (pixman + DPMS fix) |

Source: `ov/wl.go`.

## Cross-References

- `/ov:test` — parent router; `ov test wl …` is how every invocation is dispatched.
- `/ov:vnc` — VNC/RFB protocol alternative (sibling verb; TCP-based, works remotely).
- `/ov:cdp` — Chrome DevTools Protocol (sibling verb; DOM-level interaction, `--wl` flag for click, `axtree` for accessibility).
- `/ov:dbus` — D-Bus calls and desktop notifications (sibling verb under `ov test`).
- `/ov-layers:wl-tools` — Compositor-agnostic tools (wtype, wlrctl, wl-clipboard, wlr-randr, xdotool, ydotool)
- `/ov:wl-overlay` — Fullscreen overlays for recordings (title cards, lower-thirds, countdowns, highlights, fades)
- `/ov-layers:wl-overlay` — Overlay layer (gtk4-layer-shell, python3-gobject)
- `/ov-layers:wl-screenshot-grim` — Screenshot layer for sway (grim, wlr-screencopy)
- `/ov-layers:wl-screenshot-pixelflux` — Screenshot layer for selkies (pixelflux rendering pipeline)
- `/ov-layers:a11y-tools` — AT-SPI2 accessibility (python3-pyatspi, python3-gobject)
- `/ov-layers:xterm` — X11 terminal for XWayland testing
- `/ov-layers:sway-desktop` — Desktop metalayer (wl-tools + wl-screenshot-grim)
- `/ov-layers:selkies-desktop` — Desktop metalayer (wl-tools + wl-screenshot-pixelflux + a11y-tools + xterm)
