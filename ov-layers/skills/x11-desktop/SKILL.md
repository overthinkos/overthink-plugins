---
name: x11-desktop
description: |
  Base X11 desktop composition with Xorg headless, Openbox, Chrome, terminal, and file manager.
  Use when working with X11 desktop containers or comparing X11 vs Wayland stacks.
---

# x11-desktop -- Base X11 desktop composition

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `pipewire`, `openbox`, `chrome-x11`, `x11-apps` |

## Composition

```
x11-desktop
  +-- pipewire      (audio)
  +-- openbox        (Xorg headless + Openbox WM)
  +-- chrome-x11     (Chrome on X11 + DevTools)
  +-- x11-apps       (xfce4-terminal, thunar)
```

## Comparison with sway-desktop and niri-desktop

| Component | sway-desktop | niri-desktop | x11-desktop |
|-----------|-------------|-------------|------------|
| Display | Sway (Wayland) | Niri (Wayland) | Xorg headless (X11) |
| WM | Sway (tiling) | Niri (scrolling) | Openbox (floating) |
| Portals | xdg-portal-wlr | xdg-portal-niri | Not needed |
| wl-tools | grim, wlrctl | Not included | xdotool, xrandr |
| Built from | RPM | Source | RPM |

## Related Layers

- `/ov-layers:sway-desktop` -- Sway variant
- `/ov-layers:niri-desktop` -- Niri variant

## Used In Images

Not used in any current image definition. Available as a metalayer composition but no image references it yet.

## When to Use This Skill

Use when working with X11 desktop composition or comparing display server approaches.

## Author + Test References

- `/ov:layer` — layer authoring reference (tasks, vars, env_provides, tests block syntax)
- `/ov:eval` — declarative testing framework for the `tests:` block
