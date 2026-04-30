---
name: sway-desktop
description: |
  Base Sway desktop composition with audio, portals, Wayland tools, Chrome, terminal, file manager, and status bar.
  Use sway-desktop-vnc for VNC remote access.
---

# sway-desktop -- Base desktop composition (no display server)

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `pipewire`, `xdg-portal`, `wl-tools`, `wl-screenshot-grim`, `wl-overlay`, `wf-recorder`, `chrome-sway`, `xfce4-terminal`, `thunar`, `waybar`, `desktop-fonts`, `swaync`, `pavucontrol`, `tmux`, `asciinema`, `fastfetch` |
| Install files | none (pure composition) |

## Usage

Not used directly in images — use `sway-desktop-vnc` instead:

```yaml
# image.yml — VNC variant
sway-browser-vnc:
  layers:
    - sway-desktop-vnc
```

## Related Layers

- `/ov-selkies:pipewire` -- audio/media server (included)
- `/ov-selkies:xdg-portal` -- XDG desktop portal infrastructure (included)
- `/ov-selkies:wl-tools` -- Wayland automation tools: grim, wtype, wlrctl (included)
- `/ov-selkies:wl-overlay-layer` -- Fullscreen overlays via gtk4-layer-shell for recordings (included)
- `/ov-selkies:wf-recorder` -- Wayland screen recorder for desktop video (included)
- `/ov-selkies:chrome-sway` -- Chrome browser on Sway (included)
- `/ov-selkies:waybar` -- status bar and auto-tiling (included)
- `/ov-selkies:sway-desktop-vnc` -- VNC variant (adds wayvnc)
- `/ov-selkies:niri-desktop` -- Niri variant (Smithay compositor, built from source)

## Used In Images

- `/ov-selkies:sway-browser-vnc` (via `sway-desktop-vnc` metalayer)
- `/ov-openclaw:openclaw-sway-browser` (via `sway-desktop-vnc` metalayer)
- `/ov-openclaw:openclaw-ollama-sway-browser` (via `sway-desktop-vnc` metalayer)

## When to Use This Skill

Use when the user asks about:

- Desktop container composition architecture
- Base desktop without display server

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
