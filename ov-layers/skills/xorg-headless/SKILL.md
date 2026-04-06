---
name: xorg-headless
description: |
  Xorg X server with dummy video driver and libinput for headless containers.
  Provides virtual framebuffer + input device auto-detection from /dev/input.
---

# xorg-headless -- Headless Xorg with input support

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |
| Service | `xorg` (supervisord, priority 10) |
| Packages | xorg-x11-server-Xorg, xorg-x11-drv-dummy, xorg-x11-drv-libinput, xrandr, xdotool, xdpyinfo |
| Install files | `root.yml` (xorg-dummy.conf) |

## Environment Variables

| Variable | Value |
|----------|-------|
| `DISPLAY` | `:0` |

## Why Xorg (not Xvfb)

Xvfb is a virtual framebuffer with NO input driver support. Xorg with the `dummy` video driver provides a virtual framebuffer plus `xf86-input-libinput` which reads `/dev/input` and translates kernel events to X11 events.

## Configuration

The `xorg-dummy.conf` (installed to `/etc/X11/`) configures:
- **dummy** video driver with 256MB VRAM, 1920x1080@60Hz
- **libinput** auto-detection of `/dev/input/*` devices
- No VT switching (container, no TTY)

## Related Layers

- `/ov-layers:openbox` -- window manager (depends on xorg-headless)
- `/ov-layers:x11-desktop` -- desktop composition
- `/ov-layers:sway` -- Wayland alternative

## Used In Images

Not used in any current image definition. Part of the `x11-desktop` metalayer composition (dependency of `openbox`).

## When to Use This Skill

Use when working with headless X11 display, Xorg configuration, or the X11-based desktop stack.
