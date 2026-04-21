---
name: chrome-kwin
description: |
  Google Chrome running on KWin compositor with DevTools protocol.
  Use when working with Chrome in KWin desktop containers.
---

# chrome-kwin -- Chrome browser on KWin compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `kwin` |
| Layers (includes) | `chrome` |

## Usage

```yaml
# image.yml -- typically included via kwin-desktop composition
my-browser:
  layers:
    - chrome-kwin
```

## Used In Images

Part of `/ov-layers:kwin-desktop` composition.

## Related Layers

- `/ov-layers:kwin` -- compositor dependency
- `/ov-layers:chrome` -- Chrome browser and DevTools (included via layers)
- `/ov-layers:chrome-sway` -- Sway variant
- `/ov-layers:chrome-niri` -- Niri variant
- `/ov-layers:kwin-desktop` -- composition that includes chrome-kwin

## When to Use This Skill

Use when working with:

- Chrome running in KWin desktop
- Browser automation in KDE containers
- CDP (Chrome DevTools Protocol) in KWin containers

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
