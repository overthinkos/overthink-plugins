---
name: niri-apps
description: |
  Desktop applications (terminal, file manager) for Niri compositor.
  Use when working with the niri-apps layer.
---

# niri-apps -- Desktop apps for Niri

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `niri` |

## Packages

- `xfce4-terminal` (RPM) -- Terminal emulator
- `thunar` (RPM) -- File manager

## Usage

```yaml
# image.yml -- typically included via niri-desktop composition
my-desktop:
  layers:
    - niri-apps
```

**Note:** This layer exists because the existing `xfce4-terminal` and `thunar` layers depend on `sway`. `niri-apps` provides the same packages with a `niri` dependency instead.

## Related Layers

- `/ov-layers:niri` -- compositor dependency
- `/ov-layers:xfce4-terminal` -- Sway variant (depends on sway)
- `/ov-layers:thunar` -- Sway variant (depends on sway)
- `/ov-layers:niri-desktop` -- composition that includes this layer

## Used In Images

Not used in any current image definition. Part of the `niri-desktop` metalayer composition.

## When to Use This Skill

Use when working with desktop applications in Niri containers.

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
