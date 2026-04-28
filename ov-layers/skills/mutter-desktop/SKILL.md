---
name: mutter-desktop
description: |
  GNOME desktop composition with Mutter, PipeWire, XDG Portal, Chrome, gnome-terminal, and Nautilus.
  Use when working with Mutter/GNOME desktop containers.
---

# mutter-desktop -- GNOME desktop metalayer

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `pipewire`, `xdg-portal-gnome`, `chrome-mutter`, `mutter-apps` |
| Install files | none (pure composition) |

## Included Layers

- `pipewire` -- Audio/media server (PulseAudio compat)
- `xdg-portal-gnome` -- GNOME portal backend (ScreenCast, RemoteDesktop, AT-SPI2)
- `chrome-mutter` -- Chrome browser with DevTools
- `mutter-apps` -- gnome-terminal + Nautilus file manager

## Related Layers

- `/ov-layers:mutter` -- Mutter compositor
- `/ov-layers:kwin-desktop` -- KDE variant
- `/ov-layers:sway-desktop` -- Sway variant
- `/ov-layers:niri-desktop` -- Niri variant

## Used In Images

Not used in any current image definition. Available as a metalayer composition but no image references it yet.

## When to Use This Skill

Use when working with Mutter/GNOME-based desktop containers.

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
