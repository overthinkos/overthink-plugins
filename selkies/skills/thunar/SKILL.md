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
| Install files | `charly.yml`, `task:` |

## Packages

- `thunar` (RPM) -- Xfce file manager

## Usage

```yaml
# charly.yml -- typically used via sway-desktop composition
my-image:
  layers:
    - sway-desktop
```

## Used In Images

Part of the `sway-desktop` composition layer. Used transitively in all desktop images.

## Related Layers

- `/charly-selkies:sway` -- Sway compositor dependency
- `/charly-selkies:sway-desktop` -- composition that includes this layer
- `/charly-selkies:xfce4-terminal` -- terminal emulator (also in sway-desktop)

## When to Use This Skill

Use when the user asks about:

- Thunar file manager in Sway containers
- File browsing in desktop images
- Sway desktop application configuration

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
