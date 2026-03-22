---
name: sway-browser-sunshine-steam
description: |
  GPU-accelerated Sway desktop with Sunshine streaming, Chrome browser, and Steam.
  Use when working with Steam gaming via Sunshine/Moonlight on NVIDIA.
---

# sway-browser-sunshine-steam

GPU-accelerated Sway desktop with Sunshine game streaming, Chrome browser, Steam client, and gamescope. Full NVENC pipeline on NVIDIA headless with XWayland for Steam.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora-nonfree + CUDA) |
| Layers | sway-desktop-sunshine, steam |
| Platforms | linux/amd64 |
| Ports | 9222, 47984, 47989, 47990, 48010, 47998/udp, 47999/udp, 48000/udp |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 9222 | Chrome DevTools | HTTP |
| 47990 | Sunshine Web UI | HTTPS |
| 47998 | Video stream | UDP |
| 47999 | Control channel | UDP |
| 48000 | Audio stream | UDP |

## Quick Start

```bash
ov build sway-browser-sunshine-steam
ov start sway-browser-sunshine-steam
# Wait for services to start (~20s)
ov sun passwd sway-browser-sunshine-steam --generate
ov moon pair sway-browser-sunshine-steam --auto

# Connect via Moonlight, select "Desktop", log in to Steam (first time)
# Subsequent sessions: select "Steam Big Picture" in Moonlight
```

## Pre-configured Sunshine Apps

| App | Description |
|-----|-------------|
| Desktop | Full Sway desktop session |
| Steam Big Picture | Launches Steam in gamepad-friendly Big Picture mode |

## Key Layers

- `/ov-layers:sway-desktop-sunshine` — full desktop composition with Sunshine
- `/ov-layers:steam` — Steam client, gamescope, XWayland, Sunshine apps
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:sway-browser-sunshine` — same desktop but without Steam
- `/ov-images:openclaw-ollama-sway-sunshine` — AI workstation with Sunshine

## When to Use This Skill

Use when the user asks about Steam gaming via Sunshine/Moonlight, the steam-enabled desktop image, or GPU-accelerated game streaming with Steam Big Picture.
