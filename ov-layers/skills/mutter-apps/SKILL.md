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
# images.yml -- typically included via mutter-desktop composition
my-desktop:
  layers:
    - mutter-apps
```

## Related Layers

- `/ov-layers:mutter` -- compositor dependency
- `/ov-layers:kwin-apps` -- KDE variant (Konsole + Dolphin)
- `/ov-layers:niri-apps` -- Niri variant (xfce4-terminal + Thunar)
- `/ov-layers:mutter-desktop` -- composition that includes this layer

## When to Use This Skill

Use when working with desktop applications in Mutter/GNOME containers.
