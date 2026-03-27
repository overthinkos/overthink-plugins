---
name: sway-desktop-vnc
description: |
  Sway desktop with VNC remote access via wayvnc on port 5900.
  Composes sway-desktop base with wayvnc.
---

# sway-desktop-vnc -- Desktop with VNC remote access

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `sway-desktop`, `wayvnc` |
| Install files | none (pure composition) |

## NVIDIA Note

Uses gles2 on NVIDIA (same as all images). VNC screenshots may be gray (upstream wayvnc bug) — use `ov wl screenshot` instead.

## Used In Images

- `/ov-images:sway-browser-vnc`
- `/ov-images:openclaw-sway-browser`
- `/ov-images:openclaw-ollama-sway-browser`

## Related Layers

- `/ov-layers:sway-desktop` -- base desktop (composed)
- `/ov-layers:wayvnc` -- VNC server on tcp:5900 (added)
