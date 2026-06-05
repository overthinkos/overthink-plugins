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
| Install files | `candy.yml`, `task:` |

## Packages

- `xfce4-terminal` (RPM) -- Xfce terminal emulator

## Usage

```yaml
# box.yml -- typically used via sway-desktop composition
my-image:
  layers:
    - sway-desktop
```

## Used In Images

Part of the `sway-desktop` composition layer. Used transitively in all desktop images.

## Related Layers

- `/ov-selkies:sway` -- Sway compositor dependency
- `/ov-selkies:sway-desktop` -- composition that includes this layer
- `/ov-selkies:thunar` -- file manager (also in sway-desktop)

## When to Use This Skill

Use when the user asks about:

- Terminal emulator in Sway containers
- Xfce4 terminal configuration
- Sway desktop terminal access

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
