---
name: sway-desktop-vnc
description: |
  Sway desktop with VNC remote access via wayvnc on port 5900.
  Composes sway-desktop base with wayvnc. Sets pixman renderer override for NVIDIA headless.
---

# sway-desktop-vnc -- Desktop with VNC remote access

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `sway-desktop`, `wayvnc` |
| Install files | none (pure composition) |

## Renderer Override

The wayvnc layer writes `pixman` to `~/.config/sway/renderer`. The sway-wrapper reads this file and uses pixman instead of gles2 on NVIDIA headless. This is needed because wayvnc's `ext-image-copy-capture` protocol has an upstream bug with gles2 on NVIDIA.

## Used In Images

- `/ov-images:sway-browser-vnc`
- `/ov-images:sway-browser-vnc-moonlight`
- `/ov-images:openclaw-sway-browser`
- `/ov-images:openclaw-ollama-sway-browser`

## Related Layers

- `/ov-layers:sway-desktop` -- base desktop (composed)
- `/ov-layers:wayvnc` -- VNC server on tcp:5900 (added)
- `/ov-layers:sway-desktop-sunshine` -- Sunshine variant (alternative)
