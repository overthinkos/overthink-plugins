---
name: mutter
description: |
  GNOME Mutter Wayland compositor running headless inside containers with virtual monitor.
  Use when working with Mutter, GNOME desktop, or headless compositor setup.
---

# mutter -- GNOME Mutter headless Wayland compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |
| Service | `mutter` (supervisord, priority 10) |
| Install files | `layer.yml`, `root.yml`, `user.yml`, `mutter-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `XDG_RUNTIME_DIR` | `/tmp` |
| `WAYLAND_DISPLAY` | `wayland-0` |
| `XDG_CURRENT_DESKTOP` | `GNOME` |
| `XDG_SESSION_TYPE` | `wayland` |
| `EGL_LOG_LEVEL` | `fatal` |

## Packages

- `mutter` (RPM) -- GNOME Wayland compositor (has hard deps on `libeis` and `pipewire`)
- `xorg-x11-server-Xwayland` (RPM) -- XWayland support
- `mesa-dri-drivers`, `mesa-vulkan-drivers`, `libglvnd-egl`, `libglvnd-gles`, `egl-wayland` (RPM) -- GPU drivers

## Startup (mutter-wrapper)

- GPU detection: NVIDIA (CDI env vars), AMD/Intel (DRI render nodes)
- Waits for PipeWire socket (`/tmp/pipewire-0`) and D-Bus session socket
- Starts `mutter --headless --virtual-monitor 1920x1080`
- Waits for Wayland socket, symlinks to `wayland-0` if needed
- Exports `WAYLAND_DISPLAY` and `XDG_SESSION_TYPE` to D-Bus activation environment

## D-Bus Interfaces

Mutter registers native D-Bus interfaces (not Wayland protocols):
- `org.gnome.Mutter.ScreenCast` -- PipeWire-based screen capture
- `org.gnome.Mutter.RemoteDesktop` -- EIS/libei input injection
- `org.gnome.Mutter.DisplayConfig` -- Display/monitor management

This is architecturally different from KWin (Wayland protocol) and Sway (wlr-screencopy). The D-Bus interfaces work in headless mode.

## Key Difference: XDG_SESSION_TYPE

`xdg-desktop-portal-gnome` requires `XDG_SESSION_TYPE=wayland` to initialize its display server connection. Without it, ScreenCast/RemoteDesktop interfaces are not registered. This env var is normally set by gdm/gnome-session but must be set explicitly in headless containers.

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus (dependency)
- `/ov-layers:kwin` -- KDE compositor (alternative, disabled due to screencast limitation)
- `/ov-layers:sway` -- wlroots compositor (alternative)
- `/ov-layers:niri` -- Smithay compositor (alternative)
- `/ov-layers:mutter-apps` -- GNOME desktop applications
- `/ov-layers:mutter-desktop` -- full Mutter desktop composition

## When to Use This Skill

Use when working with:

- Mutter/GNOME Wayland compositor inside containers
- GNOME-based headless desktop environments
- Portal-based screen capture and input injection
