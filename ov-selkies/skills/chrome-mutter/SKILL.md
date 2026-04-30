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

- `/ov-selkies:mutter` -- compositor dependency
- `/ov-selkies:chrome` -- Chrome browser and DevTools (included via layers)
- `/ov-selkies:chrome-sway` -- Sway variant
- `/ov-selkies:chrome-kwin` -- KWin variant
- `/ov-selkies:chrome-niri` -- Niri variant
- `/ov-selkies:mutter-desktop` -- composition that includes chrome-mutter

## Used In Images

Not used in any current image definition. Part of the `mutter-desktop` metalayer composition.

## When to Use This Skill

Use when working with Chrome in Mutter/GNOME desktop containers.

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
