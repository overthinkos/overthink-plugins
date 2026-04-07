# wl-tools - Compositor-Agnostic Desktop Automation Tools

## Overview

Provides CLI tools for desktop automation — Wayland-native, X11, and clipboard. Used by the `ov wl` command. Works on all wlroots compositors (sway, labwc, niri). No daemon or special device access needed.

**Note:** Screenshots are NOT included in this layer. Use `wl-screenshot-grim` (sway) or `wl-screenshot-pixelflux` (selkies) depending on your compositor.

## Layer Definition

```yaml
rpm:
  packages:
    - wtype
    - wlrctl
    - xdotool
    - xprop
    - xwininfo
    - wl-clipboard
    - wlr-randr
    - ydotool
```

No dependencies — compositor-agnostic. Works with sway, labwc, niri, or any wlroots compositor.

## Key Properties

| Property | Value |
|----------|-------|
| Depends | None (compositor-agnostic) |
| Packages | `wtype`, `wlrctl`, `xdotool`, `xprop`, `xwininfo`, `wl-clipboard`, `wlr-randr`, `ydotool` |
| Service | None (all tools are one-shot CLI commands) |
| Ports | None |

## Tools

### Wayland Tools

| Tool | Protocol | Purpose |
|------|----------|---------|
| `wtype` | `zwp_virtual_keyboard_v1` | Keyboard input (type text, send keys, key combos with `-M`) |
| `wlrctl` | `wlr-virtual-pointer`, `wlr-foreign-toplevel-management` | Pointer control (move, click) + window management (list, focus, close, fullscreen, minimize) |
| `wl-copy` / `wl-paste` | `wlr-data-control` | Clipboard read/write |
| `wlr-randr` | `wlr-output-management` | Output resolution query and set |
| `ydotool` | `/dev/uinput` | Wayland-native mouse drag, scroll (press/release separation) |

### X11 Tools (via XWayland)

| Tool | Purpose |
|------|---------|
| `xdotool` | X11 window interaction (click, type, key, move, search, activate, scroll) |
| `xprop` | X11 window property inspection (WM_CLASS, _NET_WM_NAME, _NET_WM_PID) |
| `xwininfo` | X11 window geometry information |

All packages are in Fedora official repos.

## Included In

- `sway-desktop` metalayer (+ `wl-screenshot-grim` for screenshots)
- `selkies-desktop` metalayer (+ `wl-screenshot-pixelflux` for screenshots)

## What works where

| Tool | sway | labwc (selkies) | niri |
|------|------|-----------------|------|
| wtype | YES | YES | YES |
| wlrctl pointer | YES | YES | YES |
| wlrctl toplevel | YES | YES | YES |
| wlr-randr | YES | YES | YES |
| wl-clipboard | YES | YES | YES |
| xdotool | YES (XWayland) | YES (XWayland on-demand) | YES (XWayland) |
| ydotool | YES | YES (needs /dev/uinput) | YES |

## Used In Images

- `/ov-images:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-sway-browser` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-ollama-sway-browser` (via `sway-desktop` metalayer)
- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Cross-References

- `/ov:wl` — CLI command that uses these tools (22 subcommands)
- `/ov-layers:wl-screenshot-grim` — Screenshot layer for sway (grim)
- `/ov-layers:wl-screenshot-pixelflux` — Screenshot layer for selkies (pixelflux pipeline)
- `/ov-layers:sway-desktop` — Desktop metalayer that includes this layer
- `/ov-layers:selkies-desktop` — Desktop metalayer that includes this layer
