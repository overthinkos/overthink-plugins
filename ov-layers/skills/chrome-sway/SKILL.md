---
name: chrome-sway
description: |
  Google Chrome running on Sway compositor via exec autostart with DevTools protocol.
  Use when working with Chrome in Sway, browser automation, or CDP in desktop containers.
---

# chrome-sway -- Chrome browser on Sway compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sway` |
| Layers (includes) | `chrome` |
| Install files | `user.yml`, `chrome-sway.conf` |

## Usage

```yaml
# images.yml -- typically included via sway-desktop composition
my-browser:
  layers:
    - chrome-sway
```

## Used In Images

Part of `/ov-layers:sway-desktop` composition.

## Related Layers

- `/ov-layers:sway` -- compositor dependency
- `/ov-layers:chrome` -- Chrome browser and DevTools (included via layers)
- `/ov-layers:sway-desktop` -- composition that includes chrome-sway
- `/ov-layers:wayvnc` -- VNC access to see Chrome desktop

## When to Use This Skill

Use when the user asks about:

- Chrome running in Sway desktop
- Browser autostart in compositor
- CDP (Chrome DevTools Protocol) in desktop containers
- Chrome-sway configuration
