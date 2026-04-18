---
name: wayvnc
description: |
  WayVNC server on port tcp:5900 for remote access to Wayland desktops.
  Use when working with VNC access, remote desktop, or wayvnc configuration.
---

# wayvnc -- VNC server for Wayland compositors

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Ports | `tcp:5900` (non-HTTP protocol) |
| Service | `wayvnc` (supervisord, priority 20) |
| Install files | `tasks:`, `wayvnc-wrapper` |

## Packages

- `wayvnc` (RPM)

## Usage

```yaml
# images.yml -- typically included via sway-desktop composition
my-desktop:
  layers:
    - wayvnc
```

```bash
ov vnc screenshot my-image          # capture desktop
ov vnc passwd my-image --generate   # set up VeNCrypt/TLS auth
```

## NVIDIA Headless: Fixed via Pixman + DPMS Workaround

VNC screenshots work correctly on NVIDIA headless when used via `sway-desktop-vnc` (the standard VNC composition). Two fixes make this work:

1. **Pixman renderer** — `sway-desktop-vnc` sets `WLR_RENDERER=pixman` in its layer env, forcing software rendering. The `sway-wrapper` skips GPU auto-detection when `WLR_RENDERER` is pre-set.

2. **DPMS workaround** — wayvnc 0.9.1 gates screen capture on `zwlr_output_power_v1` mode events, but sway's headless backend never emits them. The `wayvnc-wrapper` performs a minimal VNC handshake to trigger wayvnc to bind the power manager, then `swaymsg "output HEADLESS-1 power on"` forces the missing event. Fixed in wayvnc git main (post-0.9.1) — remove workaround when Fedora ships the fix.

For images NOT using `sway-desktop-vnc` (custom sway + wayvnc setups), `ov wl screenshot` (grim) remains a reliable fallback.

### Startup Timing

The `wayvnc-wrapper` uses a two-phase wait:
1. Wait for Wayland display socket (`wayland-0`)
2. Wait for sway IPC socket (`sway-ipc.*.sock`) + 2s delay

This ensures sway has set the output resolution (default 1920x1080) before wayvnc connects.

## Used In Images

Part of `/ov-layers:sway-desktop` composition.

## Related Layers

- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:sway` -- Wayland compositor (provides display)
- `/ov-layers:sway-desktop` -- composition that includes wayvnc
- `/ov-layers:pipewire` -- sibling in desktop stack

## Related Commands

- `/ov:vnc` — VNC screenshot, click, type, key, mouse commands

## When to Use This Skill

Use when the user asks about:

- VNC server setup or configuration
- Remote desktop access to containers
- Port 5900 (tcp protocol)
- VNC password or TLS authentication
- wayvnc wrapper or configuration
