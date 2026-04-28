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
- `autostart` — Hands off to supervisord (`supervisorctl start chrome`) so the chrome-crash-listener circuit breaker supervises the browser. **Waits up to 10 s for `/tmp/supervisor.sock` and uses `supervisorctl avail | grep chrome` to confirm chrome is a defined program** before starting it; only falls through to a direct `chrome-wrapper` launch when chrome is not defined at all (minimal image variants without the chrome layer). CDP: internal 9223, external 9222 via cdp-proxy.

### autostart Chrome-duplication race (fixed in febb9bd)

The previous autostart logic was a one-liner:

```sh
if ! supervisorctl start chrome 2>/dev/null; then
    chrome-wrapper --force-renderer-accessibility --no-first-run --start-maximized &
fi
```

This had a TOCTOU race: if labwc's autostart ran before supervisord's unix socket
(`/tmp/supervisor.sock`) was reachable, the `supervisorctl start chrome` call failed
silently with stderr suppressed, and the fallback launched a background `chrome-wrapper`
**not managed by supervisord**. Then later, supervisord's own `[program:chrome]` could
also bring chrome up via a separate code path (e.g., a subsequent `supervisorctl start`
call from another script, or autorestart after a crash), leaving **two Chrome browser
mains** on the same `--user-data-dir=/home/user/.chrome-debug`. Because
`chrome-wrapper` deletes Chrome's `SingletonLock`/`SingletonSocket`/`SingletonCookie`
files at exec time (see `/ov-layers:chrome` — chrome-wrapper SingletonLock removal),
Chrome's own duplicate-detection cannot save us. The two parallel Chrome processes then
share GPU contexts, both render through the pixelflux Wayland compositor, both feed the
selkies-capture pipeline, and resource usage roughly doubles.

The race was observed in the wild on `ov-selkies-desktop-207.228.33.28` during the
pixelflux leak investigation: `ps axo comm,args | grep '^chrome ' | grep -v -- --type=`
showed two PIDs with distinct crashpad-handler-pids and identical command lines.

The fix (commit `febb9bd`) replaces the silent fallback with:

1. Wait up to 10 s for `/tmp/supervisor.sock` to appear (`for _ in $(seq 1 20); do
   [ -S /tmp/supervisor.sock ] && break; sleep 0.5; done`).
2. Use `supervisorctl avail | grep -q '^chrome\b'` to ask whether chrome is actually a
   defined program (works because `avail` lists all programs whether running or not,
   distinct from `start` which is a state-changing call).
3. If chrome is defined → `supervisorctl start chrome`. Supervisord becomes the single
   owner; the chrome-crash-listener circuit breaker handles failures.
4. If chrome is **not** defined (minimal image without the chrome layer) → fall through
   to the direct `chrome-wrapper &` launch.

The fix is purely a labwc-side hardening; chrome-wrapper's SingletonLock removal is
preserved. The combination is: chrome-wrapper still removes stale singleton state on
launch (defensive against crash residue), and the labwc autostart now refuses to launch
chrome twice. See `/ov-layers:chrome` for the SingletonLock and crash-listener side of
the story.

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

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
