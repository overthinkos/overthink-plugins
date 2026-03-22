---
name: sway-browser-sunshine-steam-heroic
description: |
  GPU-accelerated Sway desktop with Sunshine, Steam, and Heroic Games Launcher.
  Use when working with multi-store game streaming via Sunshine/Moonlight on NVIDIA.
---

# sway-browser-sunshine-steam-heroic

GPU-accelerated Sway desktop with Sunshine game streaming, Chrome browser, Steam, and Heroic Games Launcher. Full NVENC pipeline on NVIDIA headless with XWayland. Access Steam, Epic Games, GOG, and Amazon Prime Gaming via Moonlight.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora-nonfree + CUDA) |
| Layers | sway-desktop-sunshine, steam, heroic |
| Platforms | linux/amd64 |
| Ports | 9222, 47984, 47989, 47990, 48010, 47998/udp, 47999/udp, 48000/udp |
| Registry | ghcr.io/overthinkos |

## Quick Start

```bash
ov build sway-browser-sunshine-steam-heroic
ov start sway-browser-sunshine-steam-heroic
# Wait for services to start (~20s)
ov sun passwd sway-browser-sunshine-steam-heroic --generate
ov moon pair sway-browser-sunshine-steam-heroic --auto

# Connect via Moonlight — three apps available:
# - Desktop: full Sway desktop
# - Steam Big Picture: Steam in controller mode
# - Heroic Games Launcher: Epic/GOG/Amazon in fullscreen mode
```

## Pre-configured Sunshine Apps

| App | Description |
|-----|-------------|
| Desktop | Full Sway desktop session |
| Steam Big Picture | Steam in gamepad-friendly Big Picture mode |
| Heroic Games Launcher | Epic/GOG/Amazon in fullscreen controller UI |

## Key Layers

- `/ov-layers:sway-desktop-sunshine` — full desktop composition with Sunshine
- `/ov-layers:steam` — Steam client, gamescope, XWayland
- `/ov-layers:heroic` — Heroic Games Launcher, mangohud, gamemode
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:sway-browser-sunshine-steam` — same but without Heroic
- `/ov-images:sway-browser-sunshine` — Sunshine desktop without gaming

## When to Use This Skill

Use when the user asks about multi-store game streaming, Heroic + Steam together, or the full gaming desktop image via Sunshine/Moonlight.
