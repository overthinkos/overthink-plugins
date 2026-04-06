---
name: xdg-portal-gnome
description: |
  GNOME XDG Desktop Portal backend with ScreenCast, RemoteDesktop, and AT-SPI2 support.
  Use when working with GNOME portals, screen sharing, or libei input in Mutter containers.
---

# xdg-portal-gnome -- GNOME XDG Desktop Portal backend

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus`, `mutter`, `pipewire` |
| Install files | `layer.yml`, `user.yml`, `portals.conf` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `XDG_CURRENT_DESKTOP` | `GNOME` |

## Packages

- `xdg-desktop-portal` (RPM) -- Portal daemon
- `xdg-desktop-portal-gnome` (RPM) -- GNOME portal backend (ScreenCast, RemoteDesktop)
- `xdg-desktop-portal-gtk` (RPM) -- GTK fallback for file dialogs etc.
- `python3-gobject` (RPM) -- AT-SPI2 access for portal auto-accept

## Portal Configuration

`portals.conf` (installed to `~/.config/xdg-desktop-portal/portals.conf`):

```ini
[preferred]
default=gnome
org.freedesktop.impl.portal.ScreenCast=gnome
org.freedesktop.impl.portal.RemoteDesktop=gnome
```

## Key Requirement: XDG_SESSION_TYPE

The GNOME portal backend checks `XDG_SESSION_TYPE` at startup. Without `XDG_SESSION_TYPE=wayland`, it fails with `"Unsupported or missing session type ''"` and does not register ScreenCast/RemoteDesktop interfaces. This is set by the `mutter` layer.

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus (dependency)
- `/ov-layers:mutter` -- Mutter compositor (dependency)
- `/ov-layers:pipewire` -- PipeWire media server (dependency)
- `/ov-layers:xdg-portal` -- Sway variant (uses xdg-desktop-portal-wlr)
- `/ov-layers:xdg-portal-kde` -- KDE variant (uses xdg-desktop-portal-kde)
- `/ov-layers:xdg-portal-niri` -- Niri variant
- `/ov-layers:mutter-desktop` -- composition that includes this layer

## Used In Images

Not used in any current image definition. Part of the `mutter-desktop` metalayer composition.

## When to Use This Skill

Use when working with:

- XDG Desktop Portal on GNOME/Mutter
- Screen sharing in GNOME containers
- Portal-based input injection (libei/EIS)
