---
name: chrome-x11
description: |
  Google Chrome on X11 with DevTools protocol. Launched via Openbox autostart.
  Use when working with Chrome in X11 desktop containers.
---

# chrome-x11 -- Chrome browser on X11

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `openbox` |
| Layers (includes) | `chrome` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CHROME_FLAGS` | `--enable-features=VaapiVideoDecodeLinuxGL,VaapiIgnoreDriverChecks` |

Overrides the chrome layer's default flags to remove `--ozone-platform=wayland` (not needed on X11).

## Related Layers

- `/ov-selkies:openbox` -- window manager dependency
- `/ov-selkies:chrome` -- Chrome browser (included via layers)
- `/ov-selkies:chrome-sway` -- Sway variant
- `/ov-selkies:chrome-niri` -- Niri variant
- `/ov-selkies:x11-desktop` -- composition that includes this layer

## Used In Images

Not used in any current image definition. Part of the `x11-desktop` metalayer composition.

## When to Use This Skill

Use when working with Chrome in X11 desktop containers.

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
