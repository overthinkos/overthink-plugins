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
  - tmux                    # Terminal multiplexer (required by ov record)
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
- **Full `ov wl` automation:** 22 subcommands all working — screenshots (pixelflux), input (wtype, wlrctl), window management (wlrctl toplevel), clipboard (wl-copy/paste), resolution (wlr-randr), accessibility (AT-SPI2), XWayland tools (xdotool, xprop)
- **`ov cdp click --wl`:** CSS selector → Wayland pointer click (no VNC needed)
- **`ov cdp axtree`:** Chrome accessibility tree via CDP
- **Desktop video recording** via `ov record start --mode desktop` (capture bridge → H.264 → ffmpeg MP4, with optional audio)
- **Fullscreen overlays** via `ov wl overlay` (title cards, lower-thirds, countdowns, highlights, fades — rendered by compositor with true alpha transparency, no post-production needed)

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
| 8 | selkies | pixelflux `wayland-1` + WebSocket :8081 |
| 12 | labwc | Desktop on `wayland-0` (nested in wayland-1) |
| 14 | swaync | Notification daemon (on wayland-0) |
| 15 | waybar | Top panel (on wayland-0) |
| 18 | nginx | Web UI on :3000 |

## Used In Images

- `/ov-images:selkies-desktop`
- `/ov-images:selkies-desktop-nvidia`

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

- `/ov-layers:selkies` — Streaming engine (pixelflux, capture bridge, traefik)
- `/ov-layers:labwc` — Nested Wayland compositor
- `/ov-layers:chrome` — Chrome browser with CDP proxy and HTTP proxy support
- `/ov:wl` — Wayland automation (screenshots, input, windows)
- `/ov:cdp` — Chrome DevTools Protocol automation
- `/ov:record` — Desktop video recording via capture bridge
- `/ov:config` — Multi-instance deployment, tunnel, proxy env vars
- `/ov:deploy` — Tunnel configuration (deploy.yml-only, instance inheritance gap)
