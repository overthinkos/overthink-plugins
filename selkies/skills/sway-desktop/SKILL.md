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
# box.yml — VNC variant
sway-browser-vnc:
  layers:
    - sway-desktop-vnc
```

## Related Layers

- `/charly-selkies:pipewire` -- audio/media server (included)
- `/charly-selkies:xdg-portal` -- XDG desktop portal infrastructure (included)
- `/charly-selkies:wl-tools` -- Wayland automation tools: grim, wtype, wlrctl (included)
- `/charly-selkies:wl-overlay-layer` -- Fullscreen overlays via gtk4-layer-shell for recordings (included)
- `/charly-selkies:wf-recorder` -- Wayland screen recorder for desktop video (included)
- `/charly-selkies:chrome-sway` -- Chrome browser on Sway (included)
- `/charly-selkies:waybar` -- status bar and auto-tiling (included)
- `/charly-selkies:sway-desktop-vnc` -- VNC variant (adds wayvnc)

## Used In Images

- `/charly-selkies:sway-browser-vnc` (via `sway-desktop-vnc` metalayer)

## When to Use This Skill

Use when the user asks about:

- Desktop container composition architecture
- Base desktop without display server

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
