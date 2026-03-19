---
name: waybar
description: |
  Waybar status bar and sway-autotile for the Sway desktop compositor.
  Use when working with Waybar configuration, status bar, or automatic tiling.
---

# waybar -- Status bar and auto-tiling for Sway

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Service | `waybar` (supervisord, priority 15), `sway-autotile` (priority 16) |
| Install files | `user.yml`, `config.jsonc`, `style.css`, `waybar-wrapper`, `sway-autotile`, `sway-float-toggle` |

## Packages

- `waybar` (RPM)
- `dejavu-sans-mono-fonts` (RPM)
- `google-noto-sans-mono-fonts` (RPM)

## Usage

```yaml
# images.yml -- typically included via sway-desktop composition
my-desktop:
  layers:
    - waybar
```

## Used In Images

Part of `/overthink-layers:sway-desktop` composition.

## Related Layers

- `/overthink-layers:sway` -- compositor dependency
- `/overthink-layers:sway-desktop` -- composition that includes waybar

## When to Use This Skill

Use when the user asks about:

- Waybar configuration or styling
- Status bar in Sway desktop
- Automatic window tiling
- Desktop UI customization
