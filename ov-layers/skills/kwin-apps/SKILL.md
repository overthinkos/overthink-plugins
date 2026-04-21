---
name: kwin-apps
description: |
  KDE-native desktop applications (Konsole, Dolphin) for KWin compositor.
  Use when working with the kwin-apps layer.
---

# kwin-apps -- KDE-native desktop apps for KWin

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `kwin` |

## Packages

- `konsole` (RPM) -- KDE terminal emulator
- `dolphin` (RPM) -- KDE file manager

## Usage

```yaml
# image.yml -- typically included via kwin-desktop composition
my-desktop:
  layers:
    - kwin-apps
```

**Note:** Uses KDE-native apps (Konsole, Dolphin) instead of the GTK apps used in Sway/Niri stacks (xfce4-terminal, Thunar). Better integration with KWin's D-Bus services and portal infrastructure.

## Related Layers

- `/ov-layers:kwin` -- compositor dependency
- `/ov-layers:niri-apps` -- Niri variant (xfce4-terminal + thunar)
- `/ov-layers:kwin-desktop` -- composition that includes this layer

## Used In Images

Not used in any current image definition. Part of the `kwin-desktop` metalayer composition.

## When to Use This Skill

Use when working with desktop applications in KWin containers.

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
