---
name: chrome-niri
description: |
  Google Chrome running on Niri compositor with DevTools protocol.
  Use when working with Chrome in Niri desktop containers.
---

# chrome-niri -- Chrome browser on Niri compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `niri` |
| Layers (includes) | `chrome` |

## Usage

```yaml
# images.yml -- typically included via niri-desktop composition
my-browser:
  layers:
    - chrome-niri
```

## Used In Images

Part of `/ov-layers:niri-desktop` composition.

## Related Layers

- `/ov-layers:niri` -- compositor dependency
- `/ov-layers:chrome` -- Chrome browser and DevTools (included via layers)
- `/ov-layers:chrome-sway` -- Sway variant
- `/ov-layers:niri-desktop` -- composition that includes chrome-niri

## When to Use This Skill

Use when the user asks about Chrome running in Niri desktop containers.
