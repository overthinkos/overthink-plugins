---
name: xfce4-terminal
description: |
  Xfce4 terminal emulator for Sway desktop environments with sway config integration.
  Use when working with terminal emulators in Sway desktop containers.
---

# xfce4-terminal -- terminal emulator for Sway

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Install files | `layer.yml`, `tasks:` |

## Packages

- `xfce4-terminal` (RPM) -- Xfce terminal emulator

## Usage

```yaml
# images.yml -- typically used via sway-desktop composition
my-image:
  layers:
    - sway-desktop
```

## Used In Images

Part of the `sway-desktop` composition layer. Used transitively in all desktop images.

## Related Layers

- `/ov-layers:sway` -- Sway compositor dependency
- `/ov-layers:sway-desktop` -- composition that includes this layer
- `/ov-layers:thunar` -- file manager (also in sway-desktop)

## When to Use This Skill

Use when the user asks about:

- Terminal emulator in Sway containers
- Xfce4 terminal configuration
- Sway desktop terminal access
