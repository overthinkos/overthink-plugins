# Layer: selkies

Browser-accessible desktop streaming via WebRTC/WebSocket using pixelflux (Wayland capture) and pcmflux (audio).

## Architecture

Selkies acts as a **nested Wayland compositor** via pixelflux. It creates `wayland-1`, and labwc (the desktop compositor) renders into it. Pixelflux captures the output and streams it to the browser via WebSocket.

```
Browser → NGINX (:3000) → static web UI + WebSocket proxy
                                ↓
                     Selkies Python (:8081)
                          ├── pixelflux: Wayland capture → H.264/JPEG
                          ├── pcmflux: PulseAudio → Opus audio
                          └── xkbcommon: keyboard/mouse → Wayland inject
                                ↓
                     labwc (wayland-0, nested in pixelflux wayland-1)
```

## Installation

Selkies is installed from `selkies-project/selkies` commit `af1a1c2` (not the PyPI `selkies` package, which is the old GStreamer-based upstream). The LSIO fork declares `pixelflux` and `pcmflux` as direct dependencies. `av` and `cryptography` deps are stripped before install (not needed for websocket mode).

The web UI dashboard is built from the selkies repo's `addons/` directory using Node.js (npm), then nodejs is removed.

## Dependencies

- `supervisord`, `python`, `pipewire`

## Ports

| Port | Protocol | Service |
|------|----------|---------|
| 3000 | HTTP | NGINX (web UI + WebSocket proxy) |
| 8081 | WebSocket | Selkies streaming backend |

## Environment

| Variable | Value | Purpose |
|----------|-------|---------|
| `PIXELFLUX_WAYLAND` | `true` | Enable Wayland capture mode |
| `DRINODE` | `/dev/dri/renderD129` | GPU render node for pixelflux GL renderer |
| `DRI_NODE` | `/dev/dri/renderD129` | GPU render node for VAAPI encoding |
| `PULSE_SERVER` | `unix:/tmp/pulse/native` | PipeWire PulseAudio socket |

## Services (supervisord)

| Service | Priority | Purpose |
|---------|----------|---------|
| `selkies` | 8 | Streaming server (creates pixelflux wayland-1) |
| `nginx` | 18 | Web UI + WebSocket proxy |

## Key Files

- `selkies-wrapper` — GPU detection, PulseAudio null sink setup, NVRTC library path, starts selkies
- `nginx.conf` — Rootless NGINX config (temp paths in /tmp, port 3000, proxies /websockets to :8081)
- `root.yml` — Downloads selkies source, strips av/cryptography, pip installs, builds web dashboard (npm), creates NVRTC symlinks (`libnvrtc.so` → `libnvrtc.so.13` for NVENC), removes build deps (nodejs, npm, gcc, etc.)
- `pixi.toml` — Python 3.13 + pip + setuptools (setuptools provides distutils shim for Python 3.13, needed by GPUtil)

## GPU Encoding Status

| GPU | Rendering | Encoding | Status |
|-----|-----------|----------|--------|
| NVIDIA (CDI) | GL via renderD129 | NVENC attempted but fails (pixelflux compat issue with driver 590.48) | Working GL, CPU encoding fallback |
| AMD/Intel | Mesa VA-API drivers installed | VAAPI available | Untested |
| CPU | pixman fallback | x264enc / x264enc-striped / jpeg | Working |

**NVENC note:** pixelflux detects the GPU, CUDA initializes, but NVENC encoder init fails. All NVIDIA libraries load correctly (libnvidia-encode, libcuda, libnvrtc). Likely a pixelflux compatibility issue with driver 590.48. CPU x264enc at 60fps with striped mode (16 parallel stripes) provides good performance.

**Important:** Setting `SELKIES_ENCODER=x264enc-striped` locks to CPU mode. Leave encoder unset for GPU auto-detection.

## Volumes

- `selkies-config` → `~/.config/selkies`

## Security

- `shm_size: "1g"`

## Route

- `selkies.localhost:3000` (traefik)
