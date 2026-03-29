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
| Layers (composition) | `pipewire`, `xdg-portal`, `wl-tools`, `wl-screenshot-grim`, `wf-recorder`, `chrome-sway`, `xfce4-terminal`, `thunar`, `waybar`, `tmux`, `asciinema`, `fastfetch` |
| Install files | none (pure composition) |

## Usage

Not used directly in images — use `sway-desktop-vnc` instead:

```yaml
# images.yml — VNC variant
sway-browser-vnc:
  layers:
    - sway-desktop-vnc
```

## Related Layers

- `/ov-layers:pipewire` -- audio/media server (included)
- `/ov-layers:xdg-portal` -- XDG desktop portal infrastructure (included)
- `/ov-layers:wl-tools` -- Wayland automation tools: grim, wtype, wlrctl (included)
- `/ov-layers:wf-recorder` -- Wayland screen recorder for desktop video (included)
- `/ov-layers:chrome-sway` -- Chrome browser on Sway (included)
- `/ov-layers:waybar` -- status bar and auto-tiling (included)
- `/ov-layers:sway-desktop-vnc` -- VNC variant (adds wayvnc)
- `/ov-layers:niri-desktop` -- Niri variant (Smithay compositor, built from source)

## When to Use This Skill

Use when the user asks about:

- Desktop container composition architecture
- Base desktop without display server
