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

- `/ov-layers:openbox` -- window manager dependency
- `/ov-layers:chrome` -- Chrome browser (included via layers)
- `/ov-layers:chrome-sway` -- Sway variant
- `/ov-layers:chrome-niri` -- Niri variant
- `/ov-layers:x11-desktop` -- composition that includes this layer

## When to Use This Skill

Use when working with Chrome in X11 desktop containers.
