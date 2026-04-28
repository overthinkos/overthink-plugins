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
| Install files | `layer.yml`, `tasks:`, `swaync-wrapper`, `config.json`, `style.css` |

## Packages

- `SwayNotificationCenter` (RPM) -- provides `swaync`, `swaync-client`

## Service

Runs as a supervisord service (priority 14, `startsecs=2`) via `swaync-wrapper` which waits for the Wayland socket before starting. The wrapper uses `pkill -9 -x swaync` (exact name match) to kill any stale instance before exec, preventing D-Bus name conflicts on restart. `startsecs=2` gives swaync time to claim the D-Bus name before supervisord declares it running. Priority 14 ensures swaync starts after the compositor (10-12) but before waybar (15), so the notification indicator module is ready.

### D-Bus Auto-Activation Fix

The `tasks:` removes D-Bus auto-activation service files (`org.erikreider.swaync.service`, `org.erikreider.swaync.cc.service`) at build time. Without this fix, D-Bus spawns a competing swaync instance when waybar's `swaync-client -swb` queries the notification service ~2 seconds after startup. The `SystemdService=` directive in those files is intended to delegate to systemd, but since containers use supervisord (not systemd as PID 1), D-Bus falls back to `Exec=/usr/bin/swaync`, creating a second instance that steals the D-Bus bus name from the supervisord-managed one — causing the supervisord instance to exit with FATAL status.

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
# Preferred: use ov eval dbus notify (native Go D-Bus, no shell quoting issues)
ov eval dbus notify <image> "Title" "Body text"

# Alternative: use ov cmd with notification (triggers on completion)
ov cmd <image> "sleep 2 && echo done"

# Check if swaync is receiving notifications
ov eval dbus list <image> | grep Notifications

# Low-level: notify-send (requires libnotify layer)
ov cmd <image> "notify-send 'Title' 'Body text'" --no-notify

# swaync-client operations
ov cmd <image> "swaync-client -c"    # notification count
ov cmd <image> "swaync-client -C"    # clear all
ov cmd <image> "swaync-client -t"    # toggle panel
ov cmd <image> "swaync-client -d"    # toggle DnD
```

## Used In

- `/ov-layers:sway-desktop` -- via metalayer composition
- `/ov-layers:selkies-desktop` -- via metalayer composition

## Used In Images

- `/ov-images:sway-browser-vnc` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-sway-browser` (via `sway-desktop` metalayer)
- `/ov-images:openclaw-ollama-sway-browser` (via `sway-desktop` metalayer)
- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)

## Related Layers

- `/ov-layers:dbus` -- D-Bus session bus dependency
- `/ov-layers:libnotify` -- `notify-send` CLI (optional; `ov eval dbus notify` uses native Go D-Bus instead)
- `/ov-layers:waybar` -- notification bell module
- `/ov-layers:waybar-labwc` -- same notification bell module
- `/ov-layers:desktop-fonts` -- Nerd Font icons for notification bell

## When to Use This Skill

Use when the user asks about:

- Desktop notifications in containers
- SwayNotificationCenter configuration or styling
- Notification bell in waybar
- Do Not Disturb mode

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
