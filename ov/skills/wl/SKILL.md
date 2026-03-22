# WL - Wayland and X11 Desktop Automation

## Overview

`ov wl` commands provide desktop interaction inside running containers using both Wayland-native tools (`grim`, `wtype`, `wlrctl`) and X11 tools (`xdotool`, `import`). This is an alternative to `ov vnc` that doesn't require a VNC server — it runs tools directly inside the container via `exec`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Screenshot | `ov wl screenshot <image> [file]` | Capture desktop as PNG via grim |
| Click | `ov wl click <image> <x> <y>` | Click at absolute coordinates via wlrctl |
| Type text | `ov wl type <image> <text>` | Send keyboard input via wtype |
| Send key | `ov wl key <image> <key-name>` | Press a named key via wtype |
| Move mouse | `ov wl mouse <image> <x> <y>` | Move pointer to absolute coordinates |
| Status | `ov wl status <image>` | Check Wayland + X11 tool availability |
| List windows | `ov wl windows <image>` | List X11 windows via xdotool |
| Focus window | `ov wl focus <image> <title>` | Focus X11 window by title/class |

## Architecture

```
CLI command -> resolveContainer (engine + container name)
           -> exec sh -c "export WAYLAND_DISPLAY=... && <tool command>"
           -> capture stdout (screenshot) or run silently (input)
```

Uses `exec` into the container (same pattern as `ov sway`), not TCP connections (like `ov vnc`). All tools use native Wayland protocols — no daemon, no `/dev/uinput`, no VNC server required.

## Requirements

- Container must include the `wl-tools` layer (grim, wtype, wlrctl)
- Container must include the `sway` layer (Wayland compositor)
- Container must be running (`ov start` or `ov enable`)
- Included in `sway-desktop` metalayer by default

## Commands

### Screenshot
```bash
ov wl screenshot openclaw-sway-browser              # saves screenshot.png
ov wl screenshot openclaw-sway-browser desktop.png   # custom filename
ov wl screenshot openclaw-sway-browser --output HEADLESS-1  # specific output (default)
ov wl screenshot openclaw-sway-browser --region "0,0 800x600"  # region capture
```

Captures via `grim -o <output> -` which outputs PNG to stdout. The Go code captures the bytes and writes to a local file. No temp files inside the container.

### Click
```bash
ov wl click openclaw-sway-browser 960 540             # left click at center (1920x1080)
ov wl click openclaw-sway-browser 100 200 --button right  # right click
ov wl click openclaw-sway-browser 100 200 --from-cdp <tab-id>  # translate from CDP viewport
ov wl click openclaw-sway-browser 100 200 --from-sway <app_id>  # translate from sway window
```

**Absolute positioning:** `wlrctl` only supports relative pointer movement. The workaround: move far negative to clamp at (0,0), then move by target offset. Sway clamps the pointer to screen bounds, making this reliable.

### Type
```bash
ov wl type openclaw-sway-browser "hello world"    # types text via wtype
```
Supports full Unicode. Uses `zwp_virtual_keyboard_v1` Wayland protocol.

### Key
```bash
ov wl key openclaw-sway-browser Return       # press Enter
ov wl key openclaw-sway-browser Escape       # press Escape
ov wl key openclaw-sway-browser Tab          # press Tab
```

Uses `wtype -k <keyname>` with standard XKB key names. Valid keys: Return, Escape, Tab, BackSpace, Delete, Home, End, Page_Up, Page_Down, Up, Down, Left, Right, Insert, F1-F12, Shift_L, Shift_R, Control_L, Control_R, Alt_L, Alt_R, Super_L, Super_R, Meta_L, Meta_R, Caps_Lock, space.

### Mouse
```bash
ov wl mouse openclaw-sway-browser 500 300    # move pointer to (500, 300)
```

### Status
```bash
ov wl status openclaw-sway-browser
# Output:
# grim:      available
# wtype:     available
# wlrctl:    available
# output:    HEADLESS-1 1920x1080
```

## Differences from VNC Commands

| Aspect | `ov wl` (Wayland) | `ov vnc` (RFB) |
|--------|-------------------|----------------|
| Transport | exec into container | TCP port 5900 |
| Screenshot | grim (wlr-screencopy) | RFB framebuffer |
| Click | wlrctl (virtual pointer) | RFB pointer event |
| Type | wtype (virtual keyboard) | RFB key events |
| Resolution | Native compositor | VNC framebuffer |
| Daemon needed | No | wayvnc |
| NVIDIA headless | Works perfectly | Gray screen (upstream bug) |
| Remote access | No (local exec only) | Yes (TCP) |

On NVIDIA headless systems, `ov wl` is the reliable choice for screenshots and input — `ov vnc` suffers from an upstream sway/wayvnc `ext-image-copy-capture` bug that produces gray framebuffers.

Source: `ov/wl.go`.

## Cross-references

- `/ov:vnc` — VNC/RFB protocol alternative (TCP-based, works remotely)
- `/ov:cdp` — Chrome DevTools Protocol (DOM-level interaction)
- `/ov:sway` — Sway compositor control (window management)
- `/ov-layers:wl-tools` — Layer providing grim, wtype, wlrctl
- `/ov-layers:sway-desktop` — Desktop metalayer (includes wl-tools)
