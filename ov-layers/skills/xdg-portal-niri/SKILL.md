---
name: xdg-portal-niri
description: |
  XDG desktop portal integration for Niri compositor (GTK + GNOME backends).
  Use when working with portals, screen sharing, or file dialogs in Niri containers.
---

# xdg-portal-niri -- Desktop portals for Niri

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus`, `niri`, `pipewire` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `XDG_CURRENT_DESKTOP` | `niri` |

## Packages

- `xdg-desktop-portal` (RPM)
- `xdg-desktop-portal-gtk` (RPM)
- `xdg-desktop-portal-gnome` (RPM)

**Note:** Uses `xdg-desktop-portal-gnome` instead of `xdg-desktop-portal-wlr` (which is wlroots-specific). Niri handles ScreenCast portal natively.

## Related Layers

- `/ov-layers:xdg-portal` -- Sway/wlroots variant (uses xdg-desktop-portal-wlr)
- `/ov-layers:niri` -- compositor dependency
- `/ov-layers:pipewire` -- audio/media dependency
- `/ov-layers:niri-desktop` -- composition that includes this layer

## When to Use This Skill

Use when working with desktop portals, screen sharing, or file dialogs in Niri containers.
