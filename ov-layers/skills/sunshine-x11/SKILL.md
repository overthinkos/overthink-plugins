---
name: sunshine-x11
description: |
  Sunshine game streaming with X11 capture. No fake-udev, no Wayland hacks.
  Input via XTest (native X11). Screen capture via X11. All features work.
---

# sunshine-x11 -- Sunshine streaming with X11 capture

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord`, `xorg-headless`, `pipewire` |
| Ports | `tcp:47984`, `47989`, `47990`, `tcp:48010`, `udp:47998`, `udp:47999`, `udp:48000` |
| Service | `sunshine` (supervisord, priority 20) |
| Security | `devices: [/dev/uinput]` |
| Volume | `sunshine-config` -> `~/.config/sunshine` |
| Install files | `user.yml`, `sunshine-x11-wrapper` |

## What's Different from sunshine (Sway) and sunshine-niri

| Aspect | sunshine (Sway) | sunshine-niri (Niri) | sunshine-x11 |
|--------|----------------|---------------------|-------------|
| Display server | Sway (Wayland) | Niri (Wayland) | Xorg headless (X11) |
| Screen capture | wlr-screencopy | Broken (no protocol) | **X11 native** |
| Input injection | fake-udev (broken) | fake-udev (untested) | **XTest (native)** |
| fake-udev needed | Yes (287 lines) | Yes | **No** |
| NET_ADMIN cap | Yes | Yes | **No** |
| /dev/input mount | Yes | Yes | **No** |
| /run/udev tmpfs | Yes | Yes | **No** |
| Encoders | NVENC | NVENC | **NVENC (H.264, HEVC, AV1)** |

## Sunshine Config

Auto-generated on first start:
```ini
capture = x11
audio_sink = auto
origin_web_ui_allowed = lan
```

## Used In Images

Part of `/ov-layers:x11-desktop-sunshine` composition. Image: `/ov-images:sunshine-desktop-x11`.

## Related Layers

- `/ov-layers:sunshine` -- Sway variant (capture works, input broken)
- `/ov-layers:sunshine-niri` -- Niri variant (capture broken)
- `/ov-layers:xorg-headless` -- X11 display dependency
- `/ov-layers:pipewire` -- audio dependency
- `/ov:sun` -- Sunshine management commands
- `/ov:moon` -- Moonlight client protocol

## When to Use This Skill

Use when working with Sunshine streaming on X11, or comparing X11 vs Wayland streaming approaches.
