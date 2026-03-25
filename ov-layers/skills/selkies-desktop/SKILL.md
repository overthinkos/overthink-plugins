# Layer: selkies-desktop

Metalayer composing a full Selkies Wayland streaming desktop with Chrome, Waybar, and a file manager.

## Composition

```yaml
layers:
  - pipewire       # Audio (PulseAudio compat)
  - chrome         # Google Chrome with CDP on :9222
  - labwc          # Wayland compositor (nested in pixelflux)
  - waybar-labwc   # Bottom taskbar panel
  - selkies        # Streaming server (pixelflux + pcmflux + nginx)
```

## What You Get

A browser-accessible desktop at `http://localhost:3000` with:
- **labwc** Wayland compositor with server-side decorations
- **Waybar** bottom panel with Chrome/Terminal/Files buttons
- **Chrome** auto-started maximized with CDP on port 9222
- **foot** terminal (from Waybar or Super+E)
- **thunar** file manager (from Waybar)
- **pixelflux** Wayland capture → H.264 streaming at 60fps
- **pcmflux** audio capture → Opus encoding at 320kbps
- **PipeWire** audio server with PulseAudio compatibility
- **NGINX** web frontend on port 3000

## Startup Order

| Priority | Service | Creates |
|----------|---------|---------|
| 2 | dbus | D-Bus session bus |
| 5 | pipewire | Audio server |
| 8 | selkies | pixelflux `wayland-1` + WebSocket :8081 |
| 12 | labwc | Desktop on `wayland-0` (nested in wayland-1) |
| 15 | waybar | Bottom panel (on wayland-0) |
| 18 | nginx | Web UI on :3000 |
