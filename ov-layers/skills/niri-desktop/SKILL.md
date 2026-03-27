---
name: niri-desktop
description: |
  Base Niri desktop composition with audio, portals, Chrome, terminal, and file manager.
  Base desktop composition layer — no display server included.
---

# niri-desktop -- Base Niri desktop composition

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `pipewire`, `xdg-portal-niri`, `niri`, `chrome-niri`, `niri-apps` |
| Install files | none (pure composition) |

## Composition

```
niri-desktop
  ├─ pipewire          (audio/media)
  ├─ xdg-portal-niri   (desktop portals, GTK+GNOME backends)
  ├─ niri              (Smithay compositor, built from source)
  ├─ chrome-niri       (Chrome browser + DevTools)
  └─ niri-apps         (xfce4-terminal, thunar)
```

## Comparison with sway-desktop

| Component | sway-desktop | niri-desktop |
|-----------|-------------|-------------|
| Compositor | Sway (wlroots, packaged) | Niri (Smithay, built from source) |
| Portals | xdg-desktop-portal-wlr | xdg-desktop-portal-gnome |
| Status bar | waybar + sway-autotile | None (niri has built-in workspace indicators) |
| wl-tools | grim, wtype, wlrctl | Not included (wlrctl is wlroots-specific) |
| Config format | i3-style | KDL |

## Usage

```yaml
# images.yml
my-desktop:
  base: nvidia
  layers:
    - niri-desktop
```

## Related Layers

- `/ov-layers:sway-desktop` -- Sway variant
- `/ov-layers:niri` -- compositor (included)
- `/ov-layers:pipewire` -- audio (included)
- `/ov-layers:chrome-niri` -- Chrome browser (included)

## When to Use This Skill

Use when working with Niri desktop composition or comparing Niri vs Sway desktop stacks.
