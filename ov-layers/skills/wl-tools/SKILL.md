# wl-tools - Wayland Automation Tools

## Overview

Provides CLI tools for Wayland-native desktop automation: `grim` for screenshots, `wtype` for keyboard input, and `wlrctl` for pointer control. Used by the `ov wl` command. No daemon or special device access needed — all tools use native Wayland protocols directly.

## Layer Definition

```yaml
depends:
  - sway

rpm:
  packages:
    - grim
    - wtype
    - wlrctl
```

## Key Properties

| Property | Value |
|----------|-------|
| Depends | `sway` |
| Packages | `grim`, `wtype`, `wlrctl` |
| Service | None (all tools are one-shot CLI commands) |
| Ports | None |
| Devices | None (uses Wayland protocols, not uinput) |

## Tools

| Tool | Protocol | Purpose |
|------|----------|---------|
| `grim` | `wlr-screencopy` | Screenshot capture (PNG output) |
| `wtype` | `zwp_virtual_keyboard_v1` | Keyboard input (type text, send keys) |
| `wlrctl` | `zwlr_virtual_pointer_v1` | Pointer control (move, click) |

All three are packaged in Fedora official repos — no COPR or source builds needed.

## Included In

- `sway-desktop` metalayer (default for all desktop images)

## Cross-references

- `/ov:wl` — CLI command that uses these tools
- `/ov-layers:sway-desktop` — Desktop metalayer that includes this layer
- `/ov-layers:sway` — Required dependency (Wayland compositor)
