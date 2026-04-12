---
name: selkies-desktop-nvidia
description: |
  GPU-accelerated Selkies streaming desktop with NVIDIA CUDA.
  Use when working with the selkies-desktop-nvidia image.
---

# selkies-desktop-nvidia

GPU-accelerated variant of selkies-desktop using the NVIDIA base image with CUDA toolkit.

## Definition

```yaml
selkies-desktop-nvidia:
  base: nvidia
  layers:
    - agent-forwarding
    - selkies-desktop
    - dbus
    - ov
  ports:
    - "3000:3000"
    - "9222:9222"
    - "9224:9224"
  platforms:
    - linux/amd64
```

## Layers

`agent-forwarding` (gnupg + direnv + ssh-client) + `selkies-desktop` metalayer (pipewire + chrome + labwc + waybar-labwc + desktop-fonts + swaync + pavucontrol + wl-tools + wl-screenshot-pixelflux + wl-overlay + wl-record-pixelflux + a11y-tools + xterm + tmux + asciinema + fastfetch + selkies) + `dbus` + `ov`

## Ports

| Port | Service |
|------|---------|
| 3000 | Selkies web UI (Traefik HTTPS → static files + WebSocket proxy) |
| 9222 | Chrome DevTools Protocol (CDP, via socat relay) |
| 9224 | Chrome DevTools MCP (Streamable HTTP) |

## Volumes

- `chrome-data` → `~/.chrome-debug` (Chrome profile)
- `selkies-config` → `~/.config/selkies`

## Base

`nvidia` — Fedora 43 + RPM Fusion + CUDA toolkit + NVIDIA drivers.

## Difference from selkies-desktop

Same layers, but the `nvidia` base includes the full CUDA toolkit which may resolve the NVENC encoder initialization failure seen with the `fedora-nonfree` base. Use this variant for GPU-accelerated H.264 encoding via NVENC.

## Recording

Desktop video recording included via `wl-record-pixelflux` (part of selkies-desktop metalayer). With NVIDIA GPU, pixelflux-record may use NVENC for H.264 encoding (zero CPU overhead).

## Status

Not yet tested. The `fedora-nonfree` variant works with CPU encoding. This variant needs verification for NVENC hardware encoding.

## Key Layers
- `/ov-layers:selkies-desktop` — full Selkies streaming desktop metalayer
- `/ov-layers:dbus` — session bus for desktop services
- `/ov-layers:ov` — in-container ov binary

## Related Images
- `/ov-images:selkies-desktop` — CPU-encoding sibling on fedora-nonfree
- `/ov-images:nvidia` — parent base image with CUDA toolkit

## Related Commands
- `/ov:cdp` — drive Chrome inside the streaming desktop
- `/ov:wl` — interact with the labwc Wayland session
