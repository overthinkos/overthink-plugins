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
- `/ov-layers:nvidia` — GPU runtime and CDI device auto-detection (base)
- `/ov-layers:cuda` — CUDA toolkit and libraries (via nvidia base)
- `/ov-layers:dbus` — session bus for desktop services
- `/ov-layers:ov` — in-container `ov` binary (enables `ov eval dbus notify`)
- `/ov-layers:agent-forwarding` — SSH/GPG/direnv agent forwarding

## Related Images
- `/ov-images:selkies-desktop` — CPU-encoding sibling on fedora-nonfree
- `/ov-images:selkies-desktop-ov` — same streaming desktop stack + full ov toolchain (ov-full + container-nesting + golang + gh). Use this variant if you want to build images / start nested pods / launch VMs from inside the browser-accessible desktop.
- `/ov-images:selkies-desktop-bootc` — bootable VM flavor
- `/ov-images:nvidia` — parent base image with CUDA toolkit

## Related Commands
- `/ov:eval` — parent router for live-container verbs (`ov eval cdp|wl|dbus|vnc|mcp`)
- `/ov:cdp` — drive Chrome inside the streaming desktop
- `/ov:wl` — interact with the labwc Wayland session
- `/ov:dbus` — D-Bus notifications via in-container `ov` binary
- `/ov:mcp` — inherits chrome-devtools-mcp's 2 deploy-scope `mcp:` checks; use `ov eval mcp list-tools selkies-desktop-nvidia` to verify the MCP server is alive and exposing the full tool catalog.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
