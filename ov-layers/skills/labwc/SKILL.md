# Layer: labwc

Lightweight Wayland compositor (wlroots-based) for use as a nested desktop inside pixelflux's Wayland capture compositor.

## Architecture

labwc runs as a **nested compositor** inside pixelflux's `wayland-1` display. It creates its own `wayland-0` socket for desktop applications (Chrome, Waybar, foot, thunar). This is the same approach used by the LinuxServer.io Selkies base image.

- labwc connects to `wayland-1` (pixelflux) via `WAYLAND_DISPLAY=wayland-1` in the labwc-wrapper
- labwc creates `wayland-0` for its clients
- The global env `WAYLAND_DISPLAY=wayland-0` ensures apps connect to labwc, not pixelflux

## Dependencies

- `dbus`

## Packages

- `labwc` — Wayland compositor
- `foot` — Terminal emulator
- `xorg-x11-server-Xwayland` — X11 compatibility
- `thunar` — File manager

## Environment

| Variable | Value | Purpose |
|----------|-------|---------|
| `WAYLAND_DISPLAY` | `wayland-0` | labwc's client socket (global default for apps) |
| `XDG_RUNTIME_DIR` | `/tmp` | Runtime directory |

## Service (supervisord)

| Service | Priority | Purpose |
|---------|----------|---------|
| `labwc` | 12 | Desktop compositor (after selkies at priority 8) |

## Key Files

- `labwc-wrapper` — Waits for pixelflux's `wayland-1` socket, then starts labwc with `WAYLAND_DISPLAY=wayland-1` (hardcoded, not from env)
- `rc.xml` — labwc configuration: server-side decorations, maximize-all window rule, keyboard shortcuts (Alt+F4 close, Super+E terminal)
- `autostart` — Chrome auto-starts maximized with CDP on port 9222

## Window Rules

All windows open maximized (ideal for streaming desktop):
```xml
<windowRule identifier="*" serverDecoration="yes">
  <action name="Maximize" />
</windowRule>
```
