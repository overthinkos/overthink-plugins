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

## NVIDIA Support

Forces `WLR_RENDERER=pixman` (software rendering) via layer env. This ensures VNC screenshots work reliably on all hardware, including NVIDIA headless. The `wayvnc-wrapper` includes a DPMS workaround for wayvnc 0.9.1's headless power event bug (to be removed when Fedora ships a fixed version).

## Used In Images

- `/charly-selkies:sway-browser-vnc`

## Related Layers

- `/charly-selkies:sway-desktop` -- base desktop (composed)
- `/charly-selkies:wayvnc` -- VNC server on tcp:5900 (added)

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
