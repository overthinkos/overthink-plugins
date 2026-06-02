---
name: selkies-labwc-nvidia
description: |
  GPU-accelerated Selkies streaming desktop (labwc flavor) on the CachyOS NVIDIA base, with real NVENC.
  Use when working with the selkies-labwc-nvidia image.
---

# selkies-labwc-nvidia

The NVIDIA-GPU build of the **labwc flavor** (`/ov-selkies:selkies-desktop`) — the
same `selkies-desktop` metalayer on the CachyOS GPU base (`cachyos.nvidia`) with the
full CUDA toolkit, for NVENC-capable hosts. Always runs as a headless pod; the
pixelflux encoder is auto-selected per GPU at runtime (NVENC on NVIDIA, software
x264 otherwise). Lives in the `overthinkos/cachyos` submodule (`image/cachyos`); the
CPU sibling is `/ov-selkies:selkies-labwc` (in the `overthinkos/selkies` submodule),
and the KDE-Plasma GPU sibling is `selkies-kde-nvidia`.

## Definition

```yaml
selkies-labwc-nvidia:
  base: nvidia          # cachyos.nvidia — the CachyOS GPU base
  build:
    - pac
    - aur
  builder:
    pixi: ov.cuda-arch-builder
  layer:
    - agent-forwarding
    - selkies-desktop
    - dbus
    - ov
  port:
    - "3000:3000"
    - "9222:9222"
    - "9224:9224"
    - "2222:2222"
  platform:
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
| 2222 | SSH |

## Volumes

- `chrome-data` → `~/.chrome-debug` (Chrome profile)
- `selkies-config` → `~/.config/selkies`

## Base

`nvidia` resolves to `cachyos.nvidia` — the x86_64_v3-optimized CachyOS GPU base
(CachyOS + `nvidia` + `cuda` layers, CUDA toolkit + NVIDIA drivers). See
`/ov-distros:cachyos`, `/ov-distros:nvidia`, `/ov-distros:cuda`.

## REAL NVENC via the CUDA arch-builder

`builder.pixi: ov.cuda-arch-builder` selects the CUDA-equipped pixi builder, whose
`nvcc` + ffnvcodec NVENC headers let the `selkies` layer's `build.sh` compile
pixelflux's real `nvenc-sys` encoder. The stock `arch-builder` has no CUDA, so it
builds the NVENC stub — that is why the GPU images override `builder.pixi` to the
CUDA builder (the `selkies-kde-nvidia` sibling does the same). The CPU siblings
(`selkies-labwc`, `selkies-kde`) use the stock builder and rely on software x264 /
VAAPI.

## Difference from selkies-labwc (CPU sibling)

Same layers, but on a GPU base that includes the full CUDA toolkit, plus the
`builder.pixi: ov.cuda-arch-builder` override that builds the real NVENC encoder.
Use a GPU variant for hardware H.264 encoding (zero CPU overhead); the CPU sibling
`selkies-labwc` runs on the plain `cachyos` base with software x264 / VAAPI.

## AUR packages

`build: [pac, aur]` — the `selkies-desktop` metalayer composes `chrome` (AUR
`google-chrome`) + `wl-tools` (AUR `wlrctl`), so the image opts into the AUR builder.
Without `aur` in `build:`, both AUR packages would be silently dropped.

## Recording

Desktop video recording included via `wl-record-pixelflux` (part of the
`selkies-desktop` metalayer). On an NVIDIA GPU, pixelflux-record uses NVENC for H.264
encoding (zero CPU overhead).

## Key Layers
- `/ov-selkies:selkies-desktop` — full Selkies labwc streaming desktop metalayer
- `/ov-distros:nvidia` — GPU runtime and CDI device auto-detection
- `/ov-distros:cuda` — CUDA toolkit and libraries (via the CachyOS GPU base)
- `/ov-infrastructure:dbus-layer` — session bus for desktop services
- `/ov-tools:ov` — in-container `ov` binary (enables `ov eval dbus notify`)
- `/ov-distros:agent-forwarding` — SSH/GPG/direnv agent forwarding

## Related Images
- `/ov-selkies:selkies-desktop` — the labwc desktop metalayer this image is the GPU build of
- `selkies-labwc` (in the `overthinkos/selkies` submodule) — the CPU-encoding labwc sibling
- `selkies-kde-nvidia` — the KDE-Plasma GPU sibling (same `cachyos.nvidia` base + CUDA arch-builder)
- `/ov-openclaw:openclaw-desktop` — the CachyOS/CPU all-in-one: this streaming desktop stack + the openclaw-full gateway + AI CLIs + a CPU ollama + the full ov toolchain (ov-full + container-nesting + golang + gh). Use it if you want to build images / start nested pods / launch VMs from inside the browser-accessible desktop.
- `/ov-distros:cachyos` — the CachyOS base image family (owns this image's submodule)

## Related Commands
- `/ov-eval:eval` — parent router for live-container verbs (`ov eval cdp|wl|dbus|vnc|mcp`)
- `/ov-eval:cdp` — drive Chrome inside the streaming desktop
- `/ov-eval:wl` — interact with the labwc Wayland session
- `/ov-eval:dbus` — D-Bus notifications via in-container `ov` binary
- `/ov-build:ov-mcp-cmd` — inherits chrome-devtools-mcp's 2 deploy-scope `mcp:` checks; use `ov eval mcp list-tools selkies-labwc-nvidia` to verify the MCP server is alive and exposing the full tool catalog.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
