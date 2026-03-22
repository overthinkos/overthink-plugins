# wl-tools - Wayland and X11 Automation Tools

## Overview

Provides CLI tools for desktop automation — both Wayland-native and X11. Used by the `ov wl` command. No daemon or special device access needed.

## Layer Definition

```yaml
depends:
  - sway

rpm:
  packages:
    - grim
    - wtype
    - wlrctl
    - xdotool
    - xprop
    - xwininfo
```

## Key Properties

| Property | Value |
|----------|-------|
| Depends | `sway` |
| Packages | `grim`, `wtype`, `wlrctl`, `xdotool`, `xprop`, `xwininfo` |
| Service | None (all tools are one-shot CLI commands) |
| Ports | None |

## Tools

### Wayland Tools

| Tool | Protocol | Purpose |
|------|----------|---------|
| `grim` | `wlr-screencopy` | Screenshot capture (PNG output) |
| `wtype` | `zwp_virtual_keyboard_v1` | Keyboard input (type text, send keys) |
| `wlrctl` | `zwlr_virtual_pointer_v1` | Pointer control (move, click) |

### X11 Tools

| Tool | Purpose |
|------|---------|
| `xdotool` | X11 window interaction (click, type, key, move, search, activate) |
| `xprop` | X11 window property inspection |
| `xwininfo` | X11 window geometry information |

Note: `grim` captures both Wayland and XWayland window content on gles2. Dedicated X11 screenshot tools (`xwd`, `import`) don't work on XWayland rootless mode.

All packages are in Fedora official repos.

## Included In

- `sway-desktop` metalayer (default for all desktop images)

## Cross-references

- `/ov:wl` — CLI command that uses these tools
- `/ov-layers:sway-desktop` — Desktop metalayer that includes this layer
- `/ov-layers:sway` — Required dependency (Wayland compositor)
