---
name: swaync
description: |
  SwayNotificationCenter notification daemon for wlroots compositors (sway, labwc).
  Use when working with desktop notifications, notification center, or swaync configuration.
---

# swaync -- SwayNotificationCenter

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `dbus` |
| Service | `swaync` (supervisord, priority 14, startsecs=2) |
| Install files | `layer.yml`, `user.yml`, `swaync-wrapper`, `config.json`, `style.css` |

## Packages

- `SwayNotificationCenter` (RPM) -- provides `swaync`, `swaync-client`

## Service

Runs as a supervisord service (priority 14, `startsecs=2`) via `swaync-wrapper` which waits for the Wayland socket before starting. The wrapper uses `pkill -9 -x swaync` (exact name match) to kill any stale instance before exec, preventing D-Bus name conflicts on restart. `startsecs=2` gives swaync time to claim the D-Bus name before supervisord declares it running. Priority 14 ensures swaync starts after the compositor (10-12) but before waybar (15), so the notification indicator module is ready.

## Configuration

### config.json (`~/.config/swaync/config.json`)

- Notification panel on right side, overlay layer
- Widgets: inhibitors, title, dnd (Do Not Disturb), mpris (media), notifications
- Timeouts: 6s normal, 4s low, 0s critical (stays visible)
- Control center: 420px wide, top-right corner

### style.css (`~/.config/swaync/style.css`)

Catppuccin Mocha theme matching waybar:
- Color-coded notification priorities: blue (normal), red (critical), gray (low)
- MPRIS media player widget
- DnD toggle switch
- Rounded corners, semi-transparent backgrounds

## Waybar Integration

The waybar `custom/notification` module displays a notification bell icon using swaync-client:
- Click: toggle notification panel (`swaync-client -t`)
- Right-click: toggle Do Not Disturb (`swaync-client -d`)
- Icons use Nerd Font symbols (from `desktop-fonts` layer)

## Compositor Compatibility

Works with any wlroots compositor via `wlr-layer-shell` protocol:
- **sway** -- replaces mako (mako removed from sway layer)
- **labwc** -- first notification daemon (none was previously included)
- **niri** -- compatible but niri layer still includes mako (separate scope)

## Testing Notifications

```bash
# Send a test notification to the running container
podman exec ov-<image> env DBUS_SESSION_BUS_ADDRESS=unix:path=/tmp/dbus-session \
  notify-send "Title" "Body text"

# Send critical (stays visible until dismissed)
podman exec ov-<image> env DBUS_SESSION_BUS_ADDRESS=unix:path=/tmp/dbus-session \
  notify-send -u critical "Alert" "This stays visible"

# Check notification count
podman exec ov-<image> env DBUS_SESSION_BUS_ADDRESS=unix:path=/tmp/dbus-session \
  swaync-client -c

# Clear all notifications
podman exec ov-<image> env DBUS_SESSION_BUS_ADDRESS=unix:path=/tmp/dbus-session \
  swaync-client -C
```

## Used In

- `/ov-layers:sway-desktop` -- via metalayer composition
- `/ov-layers:selkies-desktop` -- via metalayer composition

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus dependency
- `/ov-layers:waybar` -- notification bell module
- `/ov-layers:waybar-labwc` -- same notification bell module
- `/ov-layers:desktop-fonts` -- Nerd Font icons for notification bell

## When to Use This Skill

Use when the user asks about:

- Desktop notifications in containers
- SwayNotificationCenter configuration or styling
- Notification bell in waybar
- Do Not Disturb mode
