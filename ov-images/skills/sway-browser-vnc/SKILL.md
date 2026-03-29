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

## Recording

Desktop video recording via wf-recorder (included in sway-desktop):

```bash
ov record start sway-browser-vnc -n demo --mode desktop
# ... interact ...
ov record stop sway-browser-vnc -n demo -o demo.mp4
```

## Related Images

None.
