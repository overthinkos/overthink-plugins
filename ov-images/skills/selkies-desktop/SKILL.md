# Image: selkies-desktop

Browser-accessible Wayland desktop streamed via Selkies/pixelflux WebSocket at `http://localhost:3000`.

## Definition

```yaml
selkies-desktop:
  base: fedora-nonfree
  layers:
    - selkies-desktop
  ports:
    - "3000:3000"
    - "9222:9222"
  platforms:
    - linux/amd64
```

## Base

`fedora-nonfree` — Fedora 43 with RPM Fusion (needed for multimedia codecs).

## Layers

`selkies-desktop` metalayer = pipewire + chrome + labwc + waybar-labwc + wl-tools + wl-screenshot-pixelflux + wl-record-pixelflux + a11y-tools + xterm + tmux + asciinema + selkies

## Ports

| Port | Service |
|------|---------|
| 3000 | Selkies web UI (NGINX → WebSocket proxy) |
| 9222 | Chrome DevTools Protocol (CDP, via socat relay) |

## Access

Open `http://localhost:3000` in a browser. No auth by default (legacy mode). The Selkies dashboard shows the labwc desktop with Chrome maximized and Waybar at the bottom.

## Build & Deploy

```bash
ov build selkies-desktop
ov start selkies-desktop
# Open http://localhost:3000
```

## GPU Support

- **Rendering:** NVIDIA GPU via CDI (`DRINODE=/dev/dri/renderD129`), AMD/Intel via Mesa
- **Encoding:** NVENC detected but currently fails with driver 590.48 (pixelflux compat issue). Falls back to CPU x264enc. VAAPI drivers installed for AMD/Intel.
- **CPU fallback:** x264enc-striped at 60fps (16 parallel stripes) — works well

## Volumes

- `chrome-data` → `~/.chrome-debug` (Chrome profile)
- `selkies-config` → `~/.config/selkies`

## Known Issues

1. **NVENC encoding fails** with NVIDIA driver 590.48 — pixelflux detects GPU, CUDA inits, but encoder init fails. CPU encoding works fine.
2. **Chrome volume permissions** — first deploy may need `podman unshare chown 1000:1000 $(podman volume inspect ov-selkies-desktop-chrome-data --format '{{.Mountpoint}}')`
3. **Audio** — PulseAudio null sinks created by selkies-wrapper. Audio streaming works but may have slight latency over WebSocket.

## Recording

Desktop video recording is built-in via `wl-record-pixelflux`:

```bash
ov record start selkies-desktop -n demo --mode desktop --audio
# ... interact with desktop ...
ov record stop selkies-desktop -n demo -o demo.mp4
```

## Verification

```bash
ov status selkies-desktop    # All services RUNNING
curl -s http://localhost:3000 # HTTP 200, Selkies dashboard HTML
# CDP: ov cdp status selkies-desktop
```
