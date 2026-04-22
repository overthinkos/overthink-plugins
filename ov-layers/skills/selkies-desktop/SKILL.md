---
name: selkies-desktop
description: |
  Metalayer composing a full Selkies Wayland streaming desktop with Chrome, Waybar, desktop automation tools, and accessibility introspection.
  Use when working with the selkies-desktop metalayer composition, labwc desktop, or browser-accessible remote desktops.
---

# Layer: selkies-desktop

Metalayer composing a full Selkies Wayland streaming desktop with Chrome, Waybar, desktop automation tools, accessibility introspection, and XWayland support.

## Composition

```yaml
layers:
  - pipewire                # Audio (PulseAudio compat)
  - chrome                  # Google Chrome with CDP on :9222, Chrome DevTools MCP on :9224
  - labwc                   # Wayland compositor (nested in pixelflux)
  - waybar-labwc            # Top status bar (Catppuccin Mocha, system monitors)
  - desktop-fonts           # JetBrains Mono + Nerd Fonts
  - swaync                  # SwayNotificationCenter (notification daemon)
  - pavucontrol             # PulseAudio volume control GUI
  - wl-tools                # Desktop automation (wtype, wlrctl, xdotool, wl-clipboard, wlr-randr)
  - wl-screenshot-pixelflux # Screenshots via selkies capture bridge
  - wl-overlay              # Fullscreen overlays via gtk4-layer-shell (for recordings)
  - wl-record-pixelflux    # Desktop video recording via selkies capture bridge
  - a11y-tools              # AT-SPI2 accessibility introspection (python3-pyatspi)
  - xterm                   # X11 terminal for XWayland testing
  - tmux                    # Terminal multiplexer (required by ov test record)
  - asciinema               # Terminal session recording
  - fastfetch               # System information display
  - selkies                 # Streaming server (pixelflux + pcmflux + nginx)
```

## What You Get

A browser-accessible desktop at `http://localhost:3000` with:
- **labwc** Wayland compositor with server-side decorations
- **Waybar** top panel with system monitors, clock, audio, and notification bell
- **Chrome** auto-started maximized with CDP on port 9222 and `--force-renderer-accessibility`
- **foot** terminal (from labwc or fuzzel)
- **thunar** file manager (from labwc)
- **SwayNotificationCenter** notification daemon with Catppuccin Mocha theme
- **pavucontrol** PulseAudio volume control (click audio module in Waybar)
- **JetBrains Mono + Nerd Fonts** for sharp monospace text and icons
- **xterm** X11 terminal (triggers XWayland on-demand when launched)
- **pixelflux** Wayland capture → H.264/JPEG streaming at 60fps
- **pcmflux** audio capture → Opus encoding at 320kbps
- **PipeWire** audio server with PulseAudio compatibility
- **NGINX** web frontend on port 3000
- **Full `ov test wl` automation:** 22 subcommands all working — screenshots (pixelflux), input (wtype, wlrctl), window management (wlrctl toplevel), clipboard (wl-copy/paste), resolution (wlr-randr), accessibility (AT-SPI2), XWayland tools (xdotool, xprop)
- **`ov test cdp click --wl`:** CSS selector → Wayland pointer click (no VNC needed)
- **`ov test cdp axtree`:** Chrome accessibility tree via CDP
- **Desktop video recording** via `ov test record start --mode desktop` (capture bridge → H.264 → ffmpeg MP4, with optional audio)
- **Fullscreen overlays** via `ov test wl overlay` (title cards, lower-thirds, countdowns, highlights, fades — rendered by compositor with true alpha transparency, no post-production needed)
- **Configurable keyboard layout** via `XKB_DEFAULT_LAYOUT` — German (de), French (fr), Nordic (no), etc. AltGr characters (@, €, \\, ~) work via direct scancode injection. See `/ov-layers:labwc`

## What Works / What Doesn't

| Feature | Status | Notes |
|---------|--------|-------|
| Screenshots (pixelflux-screenshot) | WORKS | Via capture bridge at /tmp/ov-capture.sock |
| Screenshots (grim) | BROKEN | labwc nested in pixelflux can't deliver screencopy frames |
| wtype (keyboard) | WORKS | Wayland virtual keyboard |
| wlrctl pointer (mouse) | WORKS | Move, click, double-click |
| wlrctl toplevel (windows) | WORKS | List, focus, close, fullscreen, minimize. Matches by app_id only (v0.2.2) |
| wlr-randr (resolution) | WORKS | Query and set output resolution |
| wl-clipboard | WORKS | Get/set/clear clipboard |
| xdotool (X11 windows) | WORKS | Needs an X11 app running (xterm). XWayland starts on-demand |
| xprop (X11 properties) | WORKS | Search by `--class` first, then `--name` |
| AT-SPI2 (atspi) | WORKS | Uses `/usr/bin/python3` (system Python, not pixi) |
| ydotool (drag/scroll) | WORKS | Needs `/dev/uinput` access |
| CDP click --wl | WORKS | Selector → Wayland pointer (same coordinate space) |
| CDP axtree | WORKS | Chrome accessibility tree with filtering |
| wl-overlay (overlays) | WORKS | True alpha, ~15s screenshot latency in controller mode. Instant in recordings |

## Startup Order

| Priority | Service | Creates |
|----------|---------|---------|
| 2 | dbus | D-Bus session bus |
| 5 | pipewire | Audio server |
| 8 | selkies | pixelflux `wayland-1` + WebSocket :8081 (process-wide `ScreenCapture` singleton) |
| 12 | labwc | Desktop on `wayland-0` (nested in wayland-1) |
| 14 | swaync | Notification daemon (on wayland-0) |
| 15 | waybar | Top panel (on wayland-0) |
| 18 | nginx | Web UI on :3000 |

**Chrome ownership:** Chrome is managed exclusively by supervisord, not by labwc's direct exec. labwc's autostart calls `supervisorctl start chrome` after a TOCTOU-safe `supervisorctl avail | grep chrome` check to avoid a race that could launch two Chrome processes against the same `--user-data-dir`. The fix is commit `febb9bd`; see `/ov-layers:labwc` (autostart Chrome-duplication race) for the full analysis and `/ov-layers:chrome` (Resource Caps & Circuit Breaker) for the crash-loop supervision pattern paired with the cgroup memory caps.

**Capture singleton:** the selkies process owns a single process-wide `ScreenCapture` instance. Screenshot requests (`/ov-layers:wl-screenshot-pixelflux`) and recording requests (`/ov-layers:wl-record-pixelflux`) both attach to the same capture bridge at `/tmp/ov-capture.sock` — there is never a second capture process. This is the state enforced by commit `6be85eb` after the `WaylandBackend` leak investigation; the per-frame cleanup fix in commit `7977b91` is the paired memory-management step. See `/ov-layers:selkies` (Pixelflux Memory Management) for the leak diagnosis, rollout recipe, and diagnostic commands.

## Used In Images

- `/ov-images:selkies-desktop`
- `/ov-images:selkies-desktop-nvidia`
- `/ov-images:selkies-desktop-bootc` -- Fedora 43 bootc VM with added Tailscale + KeePassXC

## Known bootc caveat — labwc ↔ pixelflux start-order race

On container images, `ENTRYPOINT=supervisord` with priority-based startup sequences the desktop tier cleanly. On bootc images, supervisord runs under a systemd user service (see `/ov-layers:bootc-config`), and a start-order race surfaces that's invisible in container mode:

- `labwc-wrapper` blocks at startup waiting for `/tmp/wayland-1` (pixelflux's socket).
- `pixelflux` is started by `selkies` (another supervisord program) which — via its own ordering — comes up alongside labwc, not before it.
- With `startsecs=2`, supervisord marks labwc RUNNING before pixelflux is ready. labwc-wrapper times out, exits status 1, supervisord restarts it. Meanwhile selkies exits too (labwc isn't up). Both programs cycle every ~15 s.

Stable services (`traefik`, `selkies-fileserver`, `chrome-devtools-mcp`, `cdp-proxy`, `sshd`, `tailscaled`, `swaync`, `waybar`, `dbus`, `pipewire`) keep running; the HTTPS selkies web endpoint on port 3000 and the MCP endpoint on 9224 both stay responsive throughout.

Fix options for a follow-up pass (none implemented yet — this layer is shared between container and bootc modes):

1. Raise labwc's `startsecs` so supervisord waits for pixelflux before declaring labwc RUNNING.
2. Add an explicit `priority:` ordering via supervisord's eventlistener hooks — a `PROCESS_STATE_RUNNING` listener on selkies that `supervisorctl start`s labwc only after pixelflux publishes its socket.
3. Have labwc-wrapper spawn pixelflux directly as a child (the container-mode start order without relying on supervisord for ordering).

Canonical worked example and diagnostic recipes: `/ov-images:selkies-desktop-bootc`.

## Multi-Instance Proxy Deployment

Deploy multiple instances with different HTTP proxies. Each instance gets a port offset:

| Offset | Web (3000) | CDP (9222) | MCP (9224) | SSH (2222) |
|--------|-----------|-----------|-----------|-----------|
| 1 | 3001 | 9231 | 9241 | 2231 |
| 2 | 3002 | 9232 | 9242 | 2232 |
| N | 300N | 923N | 924N | 223N |

**Tunnel must be in deploy.yml** for each instance. `ov config setup -i <ip>` does NOT inherit tunnel from the base entry. After config, manually add `tunnel: {provider: tailscale, private: all}` to the instance's deploy.yml entry, then re-run `ov config setup -i <ip>` to regenerate the quadlet with Tailscale serve commands. See `/ov:deploy` for details.

**Chrome 147+ CDP:** The `/json/new` endpoint requires the PUT HTTP method (not GET). Use `curl -X PUT "http://localhost:<cdp-port>/json/new?<url>"` to create new tabs programmatically.

See `/ov-images:selkies-desktop` for full multi-instance deployment examples.

## Related Skills

- `/ov-layers:selkies` — Streaming engine, Pixelflux Memory Management, ScreenCapture singleton, DRINODE auto-detection, keyboard layout support
- `/ov-layers:labwc` — Nested Wayland compositor + autostart Chrome-duplication race fix (commit `febb9bd`)
- `/ov-layers:chrome` — Chrome browser with CDP proxy, HTTP proxy support, resource caps, and crash-loop circuit breaker
- `/ov-layers:supervisord` — Event listener pattern (chrome-crash-listener) that owns Chrome's PID 1 escalation
- `/ov-layers:wl-record-pixelflux` — Desktop video recording via the shared capture singleton
- `/ov-layers:wl-screenshot-pixelflux` — Screenshots via the shared capture singleton
- `/ov-images:fedora-builder` — Builder image that compiles patched pixelflux from source (rpmfusion + build-toolchain codec devel libs)
- `/ov-images:selkies-desktop` — Image that bundles this metalayer
- `/ov:wl` — Wayland automation (screenshots, input, windows)
- `/ov:cdp` — Chrome DevTools Protocol automation
- `/ov:record` — Desktop video recording via capture bridge
- `/ov:update` — Per-instance update pattern used to roll out pixelflux/Chrome fixes
- `/ov:config` — Multi-instance deployment, resource caps, tunnel, proxy env vars, NO_PROXY auto-enrichment
- `/ov:deploy` — Tunnel configuration (deploy.yml-only, instance inheritance gap)

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
