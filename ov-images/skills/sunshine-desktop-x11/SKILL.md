---
name: sunshine-desktop-x11
description: |
  X11 desktop with Sunshine streaming on NVIDIA. All features work out of the box:
  video capture, audio, keyboard, mouse, gamepad. No fake-udev, no Wayland hacks.
---

# sunshine-desktop-x11

X11 desktop with Sunshine game streaming via Xorg headless + Openbox. Uses X11 native capture and XTest input injection — no fake-udev, no Wayland compositor hacks. All NVENC encoders (H.264, HEVC, AV1) work.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | x11-desktop-sunshine |
| Platforms | linux/amd64 |
| Ports | 9222, 47984, 47989, 47990, 48010, 47998/udp, 47999/udp, 48000/udp |

## Quick Start (Server Setup)

```bash
ov build sunshine-desktop-x11
ov start sunshine-desktop-x11
ov sun passwd sunshine-desktop-x11 --generate   # Set Web UI credentials
```

Sunshine Web UI at `https://localhost:47990`. Connect with any Moonlight client.

### CLI control (host-side, no video/input)

`ov moon` pairs the **host machine** as a GameStream control-plane client. It can list apps and start/stop sessions, but does NOT stream video, audio, or input:

```bash
ov moon pair sunshine-desktop-x11 --auto        # Pair host as control client
ov moon apps sunshine-desktop-x11               # List available apps
ov moon launch sunshine-desktop-x11 Desktop     # Pre-start app (control plane only)
```

### Full streaming test (with sway-browser-vnc-moonlight)

To test actual video/audio/input streaming, use the Moonlight GUI client inside `sway-browser-vnc-moonlight`:

```bash
ov start sway-browser-vnc-moonlight             # Moonlight client container
# VNC into sway-browser-vnc-moonlight (port 5900)
# Press Alt+M or click the Moonlight button in waybar
# Add sunshine-desktop-x11's host IP, pair via Moonlight GUI
# Connect and verify: video renders, mouse clicks work, keyboard types
```

## Architecture: Why X11?

Sunshine in Wayland containers requires fake-udev (synthetic netlink events for input device discovery) which produces erratic mouse/keyboard behavior. X11 avoids this entirely:

| Component | X11 (this image) | Sway (sway-browser-sunshine) | Niri (sunshine-desktop-niri) |
|-----------|------------------|------------------------------|------------------------------|
| Display | Xorg headless | Sway (wlroots) | Niri (Smithay) |
| Capture | X11 native | wlr-screencopy | Broken |
| Input | XTest (native) | fake-udev (erratic) | fake-udev (untested) |
| Workarounds | 0 | 12+ | 6+ |

## Key Layers

- `/ov-layers:x11-desktop-sunshine` -- full composition
- `/ov-layers:sunshine-x11` -- Sunshine with X11 capture
- `/ov-layers:xorg-headless` -- Xorg headless display server
- `/ov-layers:openbox` -- Openbox window manager

## Related Images

- `/ov-images:sunshine-desktop-niri` -- Niri variant (experimental, capture broken)
- `/ov-images:sway-browser-sunshine` -- Sway variant (capture works, input broken)
- `/ov-images:wolf` -- Container-native streaming (different architecture)

## When to Use This Skill

Use when the user asks about Sunshine desktop streaming, working Sunshine setup, or X11-based remote desktop on NVIDIA.
