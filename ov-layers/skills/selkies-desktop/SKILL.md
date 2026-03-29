# Layer: selkies-desktop

Metalayer composing a full Selkies Wayland streaming desktop with Chrome, Waybar, desktop automation tools, accessibility introspection, and XWayland support.

## Composition

```yaml
layers:
  - pipewire                # Audio (PulseAudio compat)
  - chrome                  # Google Chrome with CDP on :9222
  - labwc                   # Wayland compositor (nested in pixelflux)
  - waybar-labwc            # Bottom taskbar panel
  - wl-tools                # Desktop automation (wtype, wlrctl, xdotool, wl-clipboard, wlr-randr)
  - wl-screenshot-pixelflux # Screenshots via pixelflux rendering pipeline
  - wl-record-pixelflux    # Desktop video recording via pixelflux H.264 pipeline
  - a11y-tools              # AT-SPI2 accessibility introspection (python3-pyatspi)
  - xterm                   # X11 terminal for XWayland testing
  - tmux                    # Terminal multiplexer (required by ov record)
  - asciinema               # Terminal session recording
  - selkies                 # Streaming server (pixelflux + pcmflux + nginx)
```

## What You Get

A browser-accessible desktop at `http://localhost:3000` with:
- **labwc** Wayland compositor with server-side decorations
- **Waybar** bottom panel with Chrome/Terminal/Files buttons
- **Chrome** auto-started maximized with CDP on port 9222 and `--force-renderer-accessibility`
- **foot** terminal (from Waybar or Super+E)
- **thunar** file manager (from Waybar)
- **xterm** X11 terminal (triggers XWayland on-demand when launched)
- **pixelflux** Wayland capture → H.264/JPEG streaming at 60fps
- **pcmflux** audio capture → Opus encoding at 320kbps
- **PipeWire** audio server with PulseAudio compatibility
- **NGINX** web frontend on port 3000
- **Full `ov wl` automation:** 22 subcommands all working — screenshots (pixelflux), input (wtype, wlrctl), window management (wlrctl toplevel), clipboard (wl-copy/paste), resolution (wlr-randr), accessibility (AT-SPI2), XWayland tools (xdotool, xprop)
- **`ov cdp click --wl`:** CSS selector → Wayland pointer click (no VNC needed)
- **`ov cdp axtree`:** Chrome accessibility tree via CDP
- **Desktop video recording** via `ov record start --mode desktop` (pixelflux H.264 pipeline → MP4, with optional audio)

## What Works / What Doesn't

| Feature | Status | Notes |
|---------|--------|-------|
| Screenshots (pixelflux-screenshot) | WORKS | Taps into pixelflux rendering pipeline |
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

## Startup Order

| Priority | Service | Creates |
|----------|---------|---------|
| 2 | dbus | D-Bus session bus |
| 5 | pipewire | Audio server |
| 8 | selkies | pixelflux `wayland-1` + WebSocket :8081 |
| 12 | labwc | Desktop on `wayland-0` (nested in wayland-1) |
| 15 | waybar | Bottom panel (on wayland-0) |
| 18 | nginx | Web UI on :3000 |
