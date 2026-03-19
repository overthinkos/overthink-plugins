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

## Used In Images

Part of `/overthink-layers:sway-desktop` composition.

## Related Layers

- `/overthink-layers:supervisord` -- process manager dependency
- `/overthink-layers:sway` -- Wayland compositor (provides display)
- `/overthink-layers:sway-desktop` -- composition that includes wayvnc
- `/overthink-layers:pipewire` -- sibling in desktop stack

## When to Use This Skill

Use when the user asks about:

- VNC server setup or configuration
- Remote desktop access to containers
- Port 5900 (tcp protocol)
- VNC password or TLS authentication
- wayvnc wrapper or configuration
