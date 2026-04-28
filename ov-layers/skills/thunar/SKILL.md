---
name: thunar
description: |
  Thunar file manager for Sway desktop environments with sway config integration.
  Use when working with file management in Sway desktop containers.
---

# thunar -- Xfce file manager for Sway

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Install files | `layer.yml`, `tasks:` |

## Packages

- `thunar` (RPM) -- Xfce file manager

## Usage

```yaml
# image.yml -- typically used via sway-desktop composition
my-image:
  layers:
    - sway-desktop
```

## Used In Images

Part of the `sway-desktop` composition layer. Used transitively in all desktop images.

## Related Layers

- `/ov-layers:sway` -- Sway compositor dependency
- `/ov-layers:sway-desktop` -- composition that includes this layer
- `/ov-layers:xfce4-terminal` -- terminal emulator (also in sway-desktop)

## When to Use This Skill

Use when the user asks about:

- Thunar file manager in Sway containers
- File browsing in desktop images
- Sway desktop application configuration

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
