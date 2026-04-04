# Image: selkies-desktop

Browser-accessible Wayland desktop streamed via Selkies/pixelflux WebSocket at `https://localhost:3000` (HTTPS with self-signed Traefik certificate).

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

`selkies-desktop` metalayer = pipewire + chrome + labwc + waybar-labwc + desktop-fonts + swaync + pavucontrol + wl-tools + wl-screenshot-pixelflux + wl-record-pixelflux + a11y-tools + xterm + tmux + asciinema + fastfetch + selkies

## Ports

| Port | Service |
|------|---------|
| 3000 | Selkies web UI (Traefik HTTPS → static files + WebSocket proxy) |
| 9222 | Chrome DevTools Protocol (CDP, via socat relay) |

## Access

Open `https://localhost:3000` in a browser. Accept the self-signed certificate warning (Traefik auto-generates a cert with SANs for localhost, selkies.localhost, and ov-selkies-desktop). The Selkies dashboard shows the labwc desktop with Chrome and Waybar at the top.

HTTPS is required for the WebCodecs API (`VideoDecoder`) used by the Selkies JS client. From other containers on the same network, use `https://ov-selkies-desktop:3000`.

## Build & Deploy

```bash
ov build selkies-desktop
ov start selkies-desktop
# Open https://localhost:3000 (accept cert warning)
```

## Services

| Service | Priority | Purpose |
|---------|----------|---------|
| `selkies` | 8 | Wayland capture + H.264/Opus streaming via pixelflux |
| `traefik` | 18 | HTTPS reverse proxy on port 3000 |
| `selkies-fileserver` | 19 | Static file server for web UI |
| `labwc` | 12 | Wayland compositor (nested in pixelflux) |
| `waybar` | 15 | Status bar |
| `swaync` | 14 | Notification daemon |
| `pipewire` | 5 | Audio server |
| `dbus` | 2 | D-Bus session bus |
| `relay-9222` | — | socat CDP port relay |

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

## Screenshots and Recording

The capture bridge provides `ov wl screenshot` and `ov record` support:

```bash
# Screenshot (works with or without browser connected)
ov wl screenshot selkies-desktop screenshot.png

# Check bridge status
ov shell selkies-desktop -c "pixelflux-screenshot --status"

# Desktop video recording (with audio via PulseAudio)
ov record start selkies-desktop -n demo --mode desktop --audio
# ... interact with desktop ...
ov record stop selkies-desktop -n demo -o demo.mp4
```

The bridge auto-heals: if no valid H.264 frames are available (e.g., after browser disconnect), it reconnects as controller to restart the pipeline.

## Verification

```bash
ov status selkies-desktop              # All services RUNNING
curl -k https://localhost:3000         # HTTPS 200, Selkies dashboard HTML
ov wl screenshot selkies-desktop t.png # Screenshot via capture bridge
ov cdp status selkies-desktop          # CDP available on port 9222
```
