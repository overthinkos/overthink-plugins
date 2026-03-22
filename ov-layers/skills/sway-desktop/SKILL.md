---
name: sway-desktop
description: |
  Base Sway desktop composition with audio, portals, Wayland tools, Chrome, terminal, file manager, and status bar.
  No display server — use sway-desktop-vnc (VNC) or sway-desktop-sunshine (Sunshine) for remote access.
---

# sway-desktop -- Base desktop composition (no display server)

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `pipewire`, `xdg-portal`, `wl-tools`, `chrome-sway`, `xfce4-terminal`, `thunar`, `waybar` |
| Install files | none (pure composition) |

## Three-Tier Architecture

```
sway-desktop          = base desktop (no display server)
sway-desktop-vnc      = sway-desktop + wayvnc
sway-desktop-sunshine = sway-desktop + sunshine
```

## Usage

Not used directly in images — use `sway-desktop-vnc` or `sway-desktop-sunshine` instead:

```yaml
# images.yml — VNC variant
sway-browser-vnc:
  layers:
    - sway-desktop-vnc

# images.yml — Sunshine variant
sway-browser-sunshine:
  base: nvidia
  layers:
    - sway-desktop-sunshine
```

## Related Layers

- `/ov-layers:pipewire` -- audio/media server (included)
- `/ov-layers:xdg-portal` -- XDG desktop portal infrastructure (included)
- `/ov-layers:wl-tools` -- Wayland automation tools: grim, wtype, wlrctl (included)
- `/ov-layers:chrome-sway` -- Chrome browser on Sway (included)
- `/ov-layers:waybar` -- status bar and auto-tiling (included)
- `/ov-layers:sway-desktop-vnc` -- VNC variant (adds wayvnc)
- `/ov-layers:sway-desktop-sunshine` -- Sunshine variant (adds sunshine)

## When to Use This Skill

Use when the user asks about:

- Desktop container composition architecture
- Three-tier desktop layer separation
- Comparing VNC vs Sunshine desktop variants
- Base desktop without display server
