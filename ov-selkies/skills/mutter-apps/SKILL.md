---
name: mutter-apps
description: |
  GNOME-native desktop applications (gnome-terminal, Nautilus) for Mutter compositor.
  Use when working with the mutter-apps layer.
---

# mutter-apps -- GNOME-native desktop apps for Mutter

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `mutter` |

## Packages

- `gnome-terminal` (RPM) -- GNOME terminal emulator
- `nautilus` (RPM) -- GNOME file manager

## Usage

```yaml
# image.yml -- typically included via mutter-desktop composition
my-desktop:
  layers:
    - mutter-apps
```

## Related Layers

- `/ov-selkies:mutter` -- compositor dependency
- `/ov-selkies:kwin-apps` -- KDE variant (Konsole + Dolphin)
- `/ov-selkies:niri-apps` -- Niri variant (xfce4-terminal + Thunar)
- `/ov-selkies:mutter-desktop` -- composition that includes this layer

## Used In Images

Not used in any current image definition. Part of the `mutter-desktop` metalayer composition.

## When to Use This Skill

Use when working with desktop applications in Mutter/GNOME containers.

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
