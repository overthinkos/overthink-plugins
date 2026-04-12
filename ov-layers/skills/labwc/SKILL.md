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

## Keyboard Configuration

labwc-wrapper exports all XKB environment variables with defaults, making keyboard layout configurable at deploy time:

| Variable | Default | Purpose |
|----------|---------|---------|
| `XKB_DEFAULT_LAYOUT` | `us` | Keyboard layout (us, de, fr, gb, es, no, etc.) |
| `XKB_DEFAULT_VARIANT` | (empty) | Layout variant (dvorak, nodeadkeys, etc.) |
| `XKB_DEFAULT_MODEL` | `pc105` | Keyboard model (pc105, pc104, chromebook) |
| `XKB_DEFAULT_OPTIONS` | (empty) | XKB options (compose:ralt, caps:escape) |
| `XKB_DEFAULT_RULES` | `evdev` | XKB rules (always evdev, not configurable) |

All except RULES are declared as `env_accepts` — override via `ov config -e`:

```bash
# German QWERTZ layout
ov config selkies-desktop -e XKB_DEFAULT_LAYOUT=de

# French AZERTY with no dead keys
ov config selkies-desktop -e XKB_DEFAULT_LAYOUT=fr -e XKB_DEFAULT_VARIANT=nodeadkeys
```

The compositor and selkies input handler both read `XKB_DEFAULT_LAYOUT` from the environment, ensuring the scancode map matches the compositor's layout. See `/ov-layers:selkies` for the keyboard input pipeline details.

## Service (supervisord)

| Service | Priority | Purpose |
|---------|----------|---------|
| `labwc` | 12 | Desktop compositor (after selkies at priority 8) |

## Key Files

- `labwc-wrapper` — Waits for pixelflux's `wayland-1` socket, exports XKB_DEFAULT_* from env with defaults, then starts labwc
- `rc.xml` — labwc configuration: server-side decorations, maximize-all window rule, keyboard shortcuts (Alt+F4 close, Super+E terminal)
- `autostart` — Hands off to supervisord (`supervisorctl start chrome`) so the chrome-crash-listener circuit breaker supervises the browser. Falls back to a direct `chrome-wrapper` launch when supervisord isn't ready (images without `[program:chrome]`). CDP: internal 9223, external 9222 via cdp-proxy.

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

## Related Skills

- `/ov-layers:selkies` — pixelflux streaming engine (provides `wayland-1` that labwc connects to)
- `/ov-layers:selkies-desktop` — desktop metalayer that composes labwc + chrome + waybar + selkies
- `/ov-layers:waybar-labwc` — status bar configured for labwc
- `/ov-layers:chrome` — Chrome browser (auto-started by labwc's autostart)
- `/ov:wl` — Wayland automation commands (screenshots, input, window management)

## When to Use This Skill

**MUST be invoked** when the task involves the labwc layer, Wayland compositor configuration in selkies images, the labwc-wrapper script, window rules, or the nested compositor architecture (pixelflux → labwc → apps).
