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

## NVIDIA Headless: GPU Buffer Capture Limitation

wayvnc/neatvnc uses the `ext-image-copy-capture` Wayland protocol to capture compositor output. On NVIDIA headless systems (no physical display), this protocol cannot capture GPU-rendered buffers. Tested with wayvnc 0.9.1, neatvnc 0.9.0, NVIDIA driver 590.48 (RTX 4080 SUPER):

| Sway renderer | VNC result |
|----------------|------------|
| `pixman` (software) | Gray screen (upstream bug in sway 1.11 + wayvnc 0.9.1) |
| `gles2` | Blank screen |
| `vulkan` | Blank screen |
| `gles2` + `WLR_DRM_NO_MODIFIERS=1` | Blank screen |

**Known upstream issue (sway 1.11 / wlroots 0.19 + wayvnc 0.9.1):** Even with pixman, VNC produces gray framebuffers. wayvnc 0.9.1 uses `ext-image-copy-capture` which doesn't deliver frame data on pixman renderer. `grim` (using `wlr-screencopy`) works correctly. **Use `ov wl screenshot` as the reliable alternative on NVIDIA headless.**

The `sway-wrapper` sets `WLR_RENDERER=pixman` automatically for NVIDIA + headless.

### Startup Timing

The `wayvnc-wrapper` uses a two-phase wait:
1. Wait for Wayland display socket (`wayland-0`)
2. Wait for sway IPC socket (`sway-ipc.*.sock`) + 2s delay

This ensures sway has set the output resolution (default 1920x1080) before wayvnc connects. Without this, wayvnc would connect at the wlroots default 1280x720.

VNC resizing is enabled (no `--disable-resizing` flag) — VNC clients can request resolution changes.

See `/ov-layers:sway` for the renderer selection logic and `/ov-layers:chrome` for the Chrome GPU fallback that this requires.

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
