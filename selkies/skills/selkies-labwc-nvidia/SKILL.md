---
name: selkies-labwc-nvidia
description: |
  GPU-accelerated Selkies streaming desktop (labwc flavor) on the CachyOS NVIDIA base, with real NVENC.
  Use when working with the selkies-labwc-nvidia box.
---

# selkies-labwc-nvidia

The NVIDIA-GPU build of the **labwc flavor** (`/charly-selkies:selkies-labwc`) — the
same `selkies-desktop` metalayer on the CachyOS GPU base (`cachyos.nvidia`) with the
full CUDA toolkit, for NVENC-capable hosts. Always runs as a headless pod; the
pixelflux encoder is auto-selected per GPU at runtime (NVENC on NVIDIA, software
x264 otherwise). Lives in the `overthinkos/cachyos` submodule (`box/cachyos`); the
CPU sibling is `/charly-selkies:selkies-labwc` (now in this same `overthinkos/cachyos`
submodule), and the KDE-Plasma GPU sibling is `selkies-kde-nvidia`.

## Definition

```yaml
selkies-labwc-nvidia:
  base: nvidia          # cachyos.nvidia — the CachyOS GPU base
  build:
    - pac
    - aur
  builder:
    pixi: arch.cuda-arch-builder
  candy:
    - agent-forwarding
    - selkies-desktop
    - dbus
    - charly
  port:
    - "3000:3000"
    - "9222:9222"
    - "9224:9224"
    - "2222:2222"
  platform:
    - linux/amd64
```

## Layers

`agent-forwarding` (gnupg + direnv + ssh-client) + `selkies-desktop` metalayer (pipewire + chrome + labwc + waybar-labwc + desktop-fonts + swaync + pavucontrol + wl-tools + wl-screenshot-pixelflux + wl-overlay + wl-record-pixelflux + a11y-tools + xterm + tmux + asciinema + fastfetch + selkies) + `dbus` + `charly`

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
(CachyOS + `nvidia` + `cuda` candies, CUDA toolkit + NVIDIA drivers). See
`/charly-distros:cachyos`, `/charly-distros:nvidia`, `/charly-distros:cuda`.

## REAL NVENC via the CUDA arch-builder

`builder.pixi: arch.cuda-arch-builder` selects the CUDA-equipped pixi builder, whose
`nvcc` + ffnvcodec NVENC headers let the `selkies` candy's `build.sh` compile
pixelflux's real `nvenc-sys` encoder. The stock `arch-builder` has no CUDA, so it
builds the NVENC stub — that is why the GPU boxes override `builder.pixi` to the
CUDA builder (the `selkies-kde-nvidia` sibling does the same). The CPU siblings
(`selkies-labwc`, `selkies-kde`) use the stock builder and rely on software x264 /
VAAPI.

## Difference from selkies-labwc (CPU sibling)

Same candies, but on a GPU base that includes the full CUDA toolkit, plus the
`builder.pixi: arch.cuda-arch-builder` override that builds the real NVENC encoder.
Use a GPU variant for hardware H.264 encoding (zero CPU overhead); the CPU sibling
`selkies-labwc` runs on the plain `cachyos` base with software x264 / VAAPI.

## AUR packages

`build: [pac, aur]` — the `selkies-desktop` metalayer composes `chrome` (AUR
`google-chrome`) + `wl-tools` (AUR `wlrctl`), so the box opts into the AUR builder.
Without `aur` in `build:`, both AUR packages would be silently dropped.

## Recording

Desktop video recording included via `wl-record-pixelflux` (part of the
`selkies-desktop` metalayer). On an NVIDIA GPU, pixelflux-record uses NVENC for H.264
encoding (zero CPU overhead).

## Key Candies
- `/charly-selkies:selkies-desktop-layer` — full Selkies labwc streaming desktop metalayer
- `/charly-distros:nvidia` — GPU runtime and CDI device auto-detection
- `/charly-distros:cuda` — CUDA toolkit and libraries (via the CachyOS GPU base)
- `/charly-infrastructure:dbus-layer` — session bus for desktop services
- `/charly-tools:charly` — in-container `charly` binary (enables `charly check dbus notify`)
- `/charly-distros:agent-forwarding` — SSH/GPG/direnv agent forwarding

## Related Boxes
- `/charly-selkies:selkies-desktop-layer` — the labwc desktop metalayer this box is the GPU build of
- `selkies-labwc` (in this same `overthinkos/cachyos` submodule) — the CPU-encoding labwc sibling
- `selkies-kde-nvidia` — the KDE-Plasma GPU sibling (same `cachyos.nvidia` base + CUDA arch-builder)
- `/charly-openclaw:openclaw-desktop` — the CachyOS/CPU all-in-one: this streaming desktop stack + the openclaw-full gateway + AI CLIs + a CPU ollama + the full charly toolchain (charly + container-nesting + golang + gh). Use it if you want to build boxes / start nested pods / launch VMs from inside the browser-accessible desktop.
- `/charly-distros:cachyos` — the CachyOS base image family (owns this box's submodule)

## Related Commands
- `/charly-check:check` — parent router for the live-container check verbs (`charly check wl|dbus` host subcommands + the out-of-process `cdp:`/`vnc:`/`mcp:` verbs)
- `/charly-check:cdp` — drive Chrome inside the streaming desktop
- `/charly-check:wl` — interact with the labwc Wayland session
- `/charly-check:dbus` — D-Bus notifications via in-container `charly` binary
- `/charly-build:charly-mcp-cmd` — inherits chrome-devtools-mcp's 2 deploy-scope `mcp:` checks; use `charly check live selkies-labwc-nvidia --filter mcp` to verify the MCP server is alive and exposing the full tool catalog.

## Related

- `/charly-image:image` — image family umbrella (`candy:` image entries — those carrying `base:`/`from:` — in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — the embedded build vocabulary (distros, builders, init-systems)
