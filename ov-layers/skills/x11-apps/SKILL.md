---
name: x11-apps
description: |
  Desktop applications (terminal, file manager) for X11 containers.
  Use when working with the x11-apps layer.
---

# x11-apps -- Desktop apps for X11

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `xorg-headless` |
| Packages | xfce4-terminal, thunar |

## Related Layers

- `/ov-layers:xorg-headless` -- X11 display dependency
- `/ov-layers:niri-apps` -- Niri variant
- `/ov-layers:x11-desktop` -- composition that includes this layer

## When to Use This Skill

Use when working with desktop applications in X11 containers.
