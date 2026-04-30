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
# image.yml -- typically included via niri-desktop composition
my-browser:
  layers:
    - chrome-niri
```

## Used In Images

Part of `/ov-selkies:niri-desktop` composition.

## Related Layers

- `/ov-selkies:niri` -- compositor dependency
- `/ov-selkies:chrome` -- Chrome browser and DevTools (included via layers)
- `/ov-selkies:chrome-sway` -- Sway variant
- `/ov-selkies:niri-desktop` -- composition that includes chrome-niri

## When to Use This Skill

Use when the user asks about Chrome running in Niri desktop containers.

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
