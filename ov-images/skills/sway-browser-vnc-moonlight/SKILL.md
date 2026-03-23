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
ov start sway-browser-vnc-moonlight

# VNC into this container (port 5900), press Alt+M or click the Moonlight button in waybar
# Point Moonlight at the Sunshine server's IP, pair via the Moonlight GUI
```

## End-to-End Streaming Test

Use this image to test Sunshine servers with full video/audio/input:

```bash
# 1. Start the Sunshine server (X11 recommended — all features work)
ov start sunshine-desktop-x11
ov sun passwd sunshine-desktop-x11 --generate

# 2. Start this Moonlight client container
ov start sway-browser-vnc-moonlight

# 3. VNC into sway-browser-vnc-moonlight (port 5900)
# 4. Press Alt+M to launch Moonlight GUI
# 5. Add the Sunshine server's host/container IP
# 6. Pair via the Moonlight GUI (enter PIN shown → submit in Sunshine Web UI)
# 7. Connect and verify: video streams, mouse moves, keyboard types, audio plays
```

**Important:** The Moonlight GUI inside this container performs its **own independent pairing** with the Sunshine server. This is separate from `ov moon pair` (which pairs the host machine's CLI, not a streaming client). Only the Moonlight GUI handles actual video/audio/input streaming.

## Related Images

- `/ov-images:sway-browser-vnc` -- same without Moonlight
- `/ov-images:sunshine-desktop-x11` -- X11 Sunshine server (recommended, all features work)
- `/ov-images:sway-browser-sunshine` -- Sway Sunshine server (capture works, input erratic)
- `/ov-images:sunshine-desktop-niri` -- Niri Sunshine server (experimental, capture broken)
