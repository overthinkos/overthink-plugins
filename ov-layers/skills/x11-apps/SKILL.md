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

## Used In Images

Not used in any current image definition. Part of the `x11-desktop` metalayer composition.

## When to Use This Skill

Use when working with desktop applications in X11 containers.

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
