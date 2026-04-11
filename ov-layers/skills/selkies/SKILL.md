---
name: selkies
description: |
  Browser-accessible desktop streaming via WebSocket using pixelflux and pcmflux.
  Use when working with Selkies streaming engine, pixelflux, pcmflux, or browser-based remote desktop.
---

# Layer: selkies

Browser-accessible desktop streaming via WebSocket using pixelflux (Wayland capture) and pcmflux (audio), served over HTTPS via Traefik with a self-signed certificate.

## Architecture

Selkies acts as a **nested Wayland compositor** via pixelflux. It creates `wayland-1`, and labwc (the desktop compositor) renders into it. Pixelflux captures the output and streams it to the browser via WebSocket. Traefik terminates TLS and proxies both the static web UI and WebSocket traffic.

```
Browser → Traefik (:3000 HTTPS, self-signed cert)
               ├── / → selkies-fileserver (:3001, static web UI)
               └── /websockets → Selkies Python (:8081)
                                    ├── pixelflux: Wayland capture → H.264
                                    ├── pcmflux: PulseAudio → Opus audio
                                    ├── xkbcommon: keyboard/mouse → Wayland inject
                                    └── capture bridge → /tmp/ov-capture.sock
                                          ↓
                               labwc (wayland-0, nested in pixelflux wayland-1)
```

HTTPS is required because the Selkies web UI uses the WebCodecs API (`VideoDecoder`), which requires a secure context.

### Capture Bridge

The `selkies-capture-server` runs **inside the selkies process** as a background thread, started by `selkies-wrapper` before selkies main(). It dynamically switches between **controller** mode (starts the capture pipeline when no browser is connected) and **viewer** mode (passively receives broadcast frames alongside a browser).

This is the ONLY working capture path for screenshots and recording on selkies-desktop. The pixelflux Rust backend only supports one active `ScreenCapture` at a time, and grim doesn't work because labwc can't deliver wlr-screencopy frames when nested.

**Mode switching:**
- Starts as controller → sends SETTINGS + START_VIDEO → pipeline runs
- Browser connects → server sends KILL → bridge reconnects as viewer → receives broadcast frames
- Browser disconnects → PIPELINE_RESETTING or empty buffer → reconnects as controller
- Screenshot handler can request reconnect if no valid H.264 frames (self-healing)

**H.264 frame filtering:** The selkies server broadcasts both H.264 video (`0x04` prefix + 10-byte header) and Opus audio (`0x01 0x00` prefix) as binary WebSocket messages. The bridge filters by prefix, only buffering H.264 video frames and stripping the 10-byte selkies header before storing. Audio for recording is captured separately via PulseAudio.

**Protocol on `/tmp/ov-capture.sock`:**
- `SCREENSHOT\n` → 4-byte length + PNG data (or 0 + reason string on failure)
- `STREAM\n` → continuous (4-byte length + raw H.264 frame data)
- `STATUS\n` → 4-byte length + JSON status (`connected`, `mode`, `frames`, `seq`, `active_streams`, `last_error`)

## Installation

Selkies is installed from `selkies-project/selkies` commit `af1a1c2` (not the PyPI `selkies` package, which is the old GStreamer-based upstream). The LSIO fork declares `pixelflux` and `pcmflux` as direct dependencies. `av` and `cryptography` deps are stripped before install (not needed for websocket mode).

Build-time dependencies (gcc, nodejs, npm, python3-devel, libxkbcommon-devel) are handled in the pixi builder stage via `build.sh` — they never appear in the final image. The web UI dashboard is built in the same builder stage.

## Dependencies

- `supervisord`, `python`, `pipewire`

## Ports

| Port | Protocol | Backend Scheme | Service |
|------|----------|---------------|---------|
| 3000 | HTTPS | `https+insecure` | Traefik (web UI + WebSocket proxy, self-signed cert) |
| 8081 | WebSocket | (internal only) | Selkies streaming backend |

Port 3000 uses `https+insecure` backend scheme because Traefik terminates TLS with a self-signed certificate. When tunneled via Tailscale, this generates `tailscale serve --bg --https=3000 https+insecure://127.0.0.1:3000` — Tailscale terminates TLS on the tailnet side and proxies to the self-signed HTTPS backend. Plain `http://` proxying would get 404 from Traefik.

## Environment

| Variable | Value | Purpose |
|----------|-------|---------|
| `PIXELFLUX_WAYLAND` | `true` | Enable Wayland capture mode |
| `DRINODE` | Auto-detected | GPU render node — injected at runtime by `ov config` from first `/dev/dri/renderD*` device. See `/ov:config` |
| `DRI_NODE` | Auto-detected | Same as DRINODE — required by selkies VAAPI encoder. Override with `-e DRINODE=/dev/dri/renderDN` |
| `PULSE_SERVER` | `unix:/tmp/pulse/native` | PipeWire PulseAudio socket |
| `LANG` | `C.UTF-8` | UTF-8 locale — enables wtype to handle non-ASCII characters (ö, é, å, ñ, etc.) |

## Keyboard Layout Support

Selkies supports any keyboard layout via XKB environment variables. The compositor (labwc/sway) and the selkies input handler both read `XKB_DEFAULT_LAYOUT` from the environment. Set the layout at deploy time:

```bash
ov config selkies-desktop -e XKB_DEFAULT_LAYOUT=de    # German QWERTZ
ov config selkies-desktop -e XKB_DEFAULT_LAYOUT=fr    # French AZERTY
```

See `/ov-layers:labwc` for the full list of XKB variables (LAYOUT, VARIANT, MODEL, OPTIONS).

### Input Pipeline

The browser captures keyboard events and sends keysyms via WebSocket. The selkies Python server translates keysyms to scancodes using an xkbcommon keymap that matches the compositor's layout, then injects scancodes via pixelflux.

`build.sh` patches `input_handler.py` at build time with five fixes for generic layout support:

| Patch | What it does |
|-------|-------------|
| Env-based keymap | Reads `XKB_DEFAULT_*` from env instead of hardcoding US layout |
| Latin-1 bypass removal | Layout chars (ö, é, å, ñ) go through scancode map, not wtype |
| Level 2 scanning | AltGr characters (@, €, \\, ~) added to scancode map |
| AltGr direct injection | Injects AltGr scancode + key scancode via pixelflux (same device) |
| Euro bypass removal | € uses scancode map like all other characters |

### AltGr Characters

The browser sends AltGr as a **momentary press/release** (not held). By the time the character keysym arrives, AltGr is no longer in the server's active modifier set. The server handles this by injecting a complete AltGr+key sequence directly through pixelflux when it detects a level-2 keysym. This avoids the wtype fallback which races with pixelflux's input device (different Wayland clients, timing-sensitive).

### LANG=C.UTF-8

The `C.UTF-8` locale (built-in to glibc, no package needed) ensures `wtype` can decode non-ASCII characters in its argv. Without it, wtype fails with "Failed to deencode input argv" for characters like ö, ä, ü. This is the fallback path for characters not in the scancode map.

## Services (supervisord)

| Service | Priority | Purpose |
|---------|----------|---------|
| `selkies` | 8 | Streaming server (creates pixelflux wayland-1) |
| `traefik` | 18 | HTTPS reverse proxy on port 3000 (self-signed cert) |
| `selkies-fileserver` | 19 | Python static file server for web UI (port 3001, internal) |

## Key Files

- `selkies-wrapper` — GPU detection, PulseAudio null sink setup, NVRTC library path, starts selkies via capture server
- `selkies-capture-server` — WebSocket→Unix socket bridge with controller/viewer mode switching, H.264 frame filtering, self-healing screenshots, STATUS command
- `selkies-fileserver` — Python SPA file server with index.html fallback (serves `/usr/local/share/selkies/web/`)
- `traefik.yml` — Traefik static config (HTTPS entrypoint on :3000, self-signed cert)
- `traefik-dynamic.yml` — Path-based routing (`/websockets` → :8081, `/` → :3001) with TLS default certificate
- `build.sh` — Pixi builder stage script: pip installs selkies (C extensions), patches `input_handler.py` for generic keyboard layout support (env-based keymap, AltGr injection, bypass removal), builds web UI dashboard (npm), stages artifacts for copy to final image
- `root.yml` — Downloads Traefik binary, generates self-signed cert, installs configs (no build deps, no dnf remove)
- `pixi.toml` — Python 3.13 + pip + setuptools + libxkbcommon (C headers for builder stage)

## GPU Encoding Status

| GPU | Rendering | Encoding | Status |
|-----|-----------|----------|--------|
| NVIDIA (CDI) | GL via auto-detected renderD | NVENC attempted but fails (pixelflux compat issue with driver 590.48) | Working GL, CPU encoding fallback |
| AMD | Mesa VA-API via auto-detected renderD | VAAPI hardware H264 encoding | Working — requires correct DRINODE (auto-detected by `ov config`) |
| Intel | Mesa VA-API via auto-detected renderD | VAAPI available | Untested |
| CPU | pixman fallback | x264enc / x264enc-striped / jpeg | Working but causes flickering at high resolutions |

**DRINODE auto-detection:** `ov config` detects the first `/dev/dri/renderD*` device on the host and injects `DRINODE` and `DRI_NODE` env vars at runtime via `appendAutoDetectedEnv()`. Previously these were hardcoded to `renderD129` in `layer.yml`, causing VAAPI encoder failure on hosts with `renderD128` — the encoder fell back to CPU software encoding, which caused labwc swapchain buffer exhaustion (`No free output buffer slot`) and visible stream flickering. See `/ov-dev:go` for implementation details.

**NVENC note:** pixelflux detects the GPU, CUDA initializes, but NVENC encoder init fails. All NVIDIA libraries load correctly (libnvidia-encode, libcuda, libnvrtc). Likely a pixelflux compatibility issue with driver 590.48. CPU x264enc at 60fps with striped mode (16 parallel stripes) provides acceptable performance.

**Important:** Setting `SELKIES_ENCODER=x264enc-striped` locks to CPU mode. Leave encoder unset for GPU auto-detection.

## Volumes

- `selkies-config` → `~/.config/selkies`

## Used In Images

- `/ov-images:selkies-desktop`
- `/ov-images:selkies-desktop-nvidia`

## Related Commands

- `/ov:wl` — Wayland automation (screenshot via capture bridge, input, windows)
- `/ov:cdp` — Chrome DevTools and SPA bridge (click, type, key-combo through remote desktop)
- `/ov:record` — Desktop video recording via capture bridge
- `/ov-layers:selkies-desktop` — Desktop metalayer composing selkies with Chrome, labwc, Waybar

## Security

- `shm_size: "1g"`
