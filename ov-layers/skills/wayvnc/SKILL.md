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
| Install files | `user.yml`, `wayvnc-wrapper` |

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

## NVIDIA Headless: VNC Screenshot Limitation

wayvnc's `ext-image-copy-capture` protocol produces gray/blank screenshots on NVIDIA headless (upstream bug in sway 1.11 / wlroots 0.19 + wayvnc 0.9.1). VNC remote viewing (connecting a VNC client) still works, but `ov vnc screenshot` produces gray images.

**For screenshots on NVIDIA headless, use `ov wl screenshot`** (grim via `wlr-screencopy`). Works correctly with gles2 on NVIDIA, capturing both Wayland and XWayland window content.

All images use `gles2` renderer on NVIDIA (hardware auto-detect in sway-wrapper). No renderer overrides.

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

## When to Use This Skill

Use when the user asks about:

- VNC server setup or configuration
- Remote desktop access to containers
- Port 5900 (tcp protocol)
- VNC password or TLS authentication
- wayvnc wrapper or configuration
