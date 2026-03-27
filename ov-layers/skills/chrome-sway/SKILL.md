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

## Chrome Lifecycle in Sway

Chrome is launched by Sway's `exec` directive (autostart) via `chrome-wrapper`. It is **not** a supervisord service -- Sway owns the Chrome process. This means:

- **Autostart**: Chrome starts when Sway starts (via `exec chrome-wrapper` in sway config).
- **Crashes/exits**: Chrome does not auto-restart. If Chrome exits, it must be relaunched manually.
- **Manual restart**: Use `ov wl sway exec <image> chrome-wrapper` to relaunch Chrome from outside the container. Do **not** use `ov shell` with bare `swaymsg` -- the shell may lack the correct `SWAYSOCK` path.
- **On-demand launch**: The `browser-open` helper auto-launches Chrome if it's not running (see `/ov-layers:chrome` for details).

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
