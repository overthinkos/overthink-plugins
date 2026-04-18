---
name: xdg-portal-kde
description: |
  KDE XDG Desktop Portal backend with ScreenCast and RemoteDesktop support.
  Use when working with KDE portals, screen sharing, or libei input in KWin containers.
---

# xdg-portal-kde -- KDE XDG Desktop Portal backend

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus`, `kwin`, `pipewire` |
| Install files | `layer.yml`, `tasks:`, `portals.conf` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `XDG_CURRENT_DESKTOP` | `KDE` |

## Packages

- `xdg-desktop-portal` (RPM) -- Portal daemon
- `xdg-desktop-portal-kde` (RPM) -- KDE portal backend (ScreenCast, RemoteDesktop, FileChooser)

## Portal Configuration

`portals.conf` (installed to `~/.config/xdg-desktop-portal/portals.conf`):

```ini
[preferred]
default=kde
org.freedesktop.impl.portal.ScreenCast=kde
org.freedesktop.impl.portal.RemoteDesktop=kde
```

Routes ScreenCast and RemoteDesktop requests to the KDE backend, which supports both PipeWire screen capture and libei input injection.

## Portal Capabilities

The KDE portal backend provides:

- **ScreenCast** -- PipeWire-based screen capture via `zkde_screencast_unstable_v1`
- **RemoteDesktop** -- Input injection via libei/EIS (`ConnectToEIS` method)
- **FileChooser**, **Screenshot**, **Notification**, **GlobalShortcuts**, **InputCapture**, **Clipboard**

## Known Limitation

The `zkde_screencast_unstable_v1` Wayland protocol is not exposed by KWin's `--virtual` backend (as of KDE 6.6.3). Portal ScreenCast/RemoteDesktop sessions fail with response code 2 in headless mode.

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus (dependency)
- `/ov-layers:kwin` -- KWin compositor (dependency)
- `/ov-layers:pipewire` -- PipeWire media server (dependency)
- `/ov-layers:xdg-portal` -- Sway variant (uses xdg-desktop-portal-wlr)
- `/ov-layers:xdg-portal-niri` -- Niri variant (uses xdg-desktop-portal-gnome)
- `/ov-layers:kwin-desktop` -- composition that includes this layer

## Used In Images

Not used in any current image definition. Part of the `kwin-desktop` metalayer composition.

## When to Use This Skill

Use when working with:

- XDG Desktop Portal on KDE/KWin
- Screen sharing in KDE containers
- Portal-based input injection (libei/EIS)
- KDE portal configuration
