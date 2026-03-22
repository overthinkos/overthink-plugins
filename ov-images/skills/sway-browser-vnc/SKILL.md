---
name: sway-browser-vnc
description: |
  Minimal Sway desktop with VNC remote access and Chrome browser.
  Use when working with VNC desktop containers or testing the sway-desktop-vnc composition.
---

# sway-browser-vnc

Minimal Sway desktop with VNC (wayvnc on port 5900) and Chrome (CDP on port 9222).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | sway-desktop-vnc |
| Ports | 5900, 9222 |

## Quick Start

```bash
ov build sway-browser-vnc
ov start sway-browser-vnc
ov vnc status sway-browser-vnc
ov wl screenshot sway-browser-vnc screenshot.png
```

## Related Images

- `/ov-images:sway-browser-vnc-moonlight` -- same + Moonlight client
- `/ov-images:sway-browser-sunshine` -- Sunshine variant (GPU streaming)
