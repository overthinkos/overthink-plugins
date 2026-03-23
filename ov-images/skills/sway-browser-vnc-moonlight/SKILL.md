---
name: sway-browser-vnc-moonlight
description: |
  Sway desktop with VNC and Moonlight client for testing Sunshine streaming.
  Use when debugging Sunshine/Moonlight connectivity from within a container.
---

# sway-browser-vnc-moonlight

Sway desktop with VNC remote access, Chrome browser, and Moonlight game streaming client. Use to test and debug Sunshine streaming from within a container — VNC into this image and launch Moonlight (Alt+M) to connect to a Sunshine server.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | sway-desktop-vnc, moonlight |
| Ports | 5900, 9222 |

## Waybar Moonlight Button

The moonlight layer patches waybar to add a **purple Moonlight launcher button** in the status bar. Click it (or press Alt+M) to launch Moonlight. The button appears in `modules-left` next to the terminal button, styled in Catppuccin Mocha purple.

## Quick Start

```bash
ov build sway-browser-vnc-moonlight
ov start sway-browser-vnc-moonlight -p 5901:5900 -p 9227:9222

# VNC into this container, press Alt+M or click the Moonlight button in waybar
# Point Moonlight at the Sunshine container's IP
```

## Related Images

- `/ov-images:sway-browser-vnc` -- same without Moonlight
- `/ov-images:sway-browser-sunshine` -- the Sunshine server to connect to
