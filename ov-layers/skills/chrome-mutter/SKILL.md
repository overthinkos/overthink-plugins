---
name: chrome-mutter
description: |
  Google Chrome running on Mutter compositor with DevTools protocol.
  Use when working with Chrome in Mutter/GNOME desktop containers.
---

# chrome-mutter -- Chrome browser on Mutter compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `mutter` |
| Layers (includes) | `chrome` |

## Usage

```yaml
# image.yml -- typically included via mutter-desktop composition
my-browser:
  layers:
    - chrome-mutter
```

## Related Layers

- `/ov-layers:mutter` -- compositor dependency
- `/ov-layers:chrome` -- Chrome browser and DevTools (included via layers)
- `/ov-layers:chrome-sway` -- Sway variant
- `/ov-layers:chrome-kwin` -- KWin variant
- `/ov-layers:chrome-niri` -- Niri variant
- `/ov-layers:mutter-desktop` -- composition that includes chrome-mutter

## Used In Images

Not used in any current image definition. Part of the `mutter-desktop` metalayer composition.

## When to Use This Skill

Use when working with Chrome in Mutter/GNOME desktop containers.

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
