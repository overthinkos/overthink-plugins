---
name: kwin
description: |
  KWin Wayland compositor running headless inside containers with virtual backend.
  Use when working with KWin, KDE desktop, or headless compositor setup.
---

# kwin -- KWin headless Wayland compositor

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |
| Service | `kwin` (supervisord, priority 10) |
| Install files | `layer.yml`, `root.yml`, `user.yml`, `kwin-wrapper` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `XDG_RUNTIME_DIR` | `/tmp` |
| `WAYLAND_DISPLAY` | `wayland-0` |
| `XDG_CURRENT_DESKTOP` | `KDE` |
| `EGL_LOG_LEVEL` | `fatal` |

## Packages

- `kwin-wayland` (RPM) -- KDE Wayland compositor
- `kpipewire` (RPM) -- KDE PipeWire integration (required for screencast plugin)
- `xorg-x11-server-Xwayland` (RPM) -- XWayland support
- `mesa-dri-drivers`, `mesa-vulkan-drivers`, `libglvnd-egl`, `libglvnd-gles`, `egl-wayland` (RPM) -- GPU drivers

## Startup (kwin-wrapper)

- GPU detection: NVIDIA (CDI env vars), AMD/Intel (DRI render nodes)
- Waits for PipeWire socket (`/tmp/pipewire-0`) before starting KWin
- Starts `kwin_wayland --virtual --width 1920 --height 1080`
- Waits for Wayland socket, symlinks to `wayland-0` if needed
- Exports `WAYLAND_DISPLAY` to D-Bus activation environment

## KWin Plugins

With `kpipewire` installed, KWin loads: `screencast`, `eis` (libei input), `screenshot`, `nightlight`, `buttonsrebind`.

## Known Limitation

KWin's `--virtual` backend does NOT expose `zkde_screencast_unstable_v1` Wayland protocol, even with the screencast plugin loaded. This means `xdg-desktop-portal-kde` ScreenCast/RemoteDesktop portal sessions fail with response code 2. Portal-based capture does not work in headless mode as of KDE 6.6.3.

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus (dependency)
- `/ov-layers:sway` -- wlroots compositor (alternative)
- `/ov-layers:niri` -- Smithay compositor (alternative)
- `/ov-layers:xorg-headless` -- X11 display server (alternative)
- `/ov-layers:kwin-apps` -- KDE desktop applications
- `/ov-layers:kwin-desktop` -- full KWin desktop composition

## Used In Images

Not used in any current image definition. Part of the `kwin-desktop` metalayer composition.

## When to Use This Skill

Use when working with:

- KWin Wayland compositor inside containers
- KDE-based desktop environments
- KWin headless/virtual mode configuration
