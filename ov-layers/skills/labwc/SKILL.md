---
name: labwc
description: |
  Lightweight Wayland compositor (wlroots-based) for nested desktop inside pixelflux.
  MUST be invoked when working with: the labwc layer, Wayland compositor config in selkies images, or labwc-wrapper.
---

# labwc -- Nested Wayland compositor for Selkies desktop

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

## Used In Images

- `/ov-images:selkies-desktop` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-nvidia` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-hermes` (via `selkies-desktop` metalayer)
- `/ov-images:selkies-desktop-hermes-jupyter` (via `selkies-desktop` metalayer)

## Related Skills

- `/ov-layers:selkies` — pixelflux streaming engine (provides `wayland-1` that labwc connects to)
- `/ov-layers:selkies-desktop` — desktop metalayer that composes labwc + chrome + waybar + selkies
- `/ov-layers:waybar-labwc` — status bar configured for labwc
- `/ov-layers:chrome` — Chrome browser (auto-started by labwc's autostart)
- `/ov:wl` — Wayland automation commands (screenshots, input, window management)

## When to Use This Skill

**MUST be invoked** when the task involves the labwc layer, Wayland compositor configuration in selkies images, the labwc-wrapper script, window rules, or the nested compositor architecture (pixelflux → labwc → apps).
