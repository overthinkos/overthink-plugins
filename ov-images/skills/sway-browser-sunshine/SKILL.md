---
name: sway-browser-sunshine
description: |
  GPU-accelerated Sway desktop with Sunshine streaming and Chrome browser.
  Lightweight test image for Sunshine game streaming on NVIDIA.
---

# sway-browser-sunshine

GPU-accelerated Sway desktop with Sunshine game streaming and Chrome browser. Uses NVENC for video encoding and gles2 for compositing on NVIDIA headless.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | sway-desktop-sunshine |
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
ov build sway-browser-sunshine
ov start sway-browser-sunshine
# Wait for services to start (~20s)
ov sun passwd sway-browser-sunshine --generate   # Set Sunshine Web UI credentials
ov moon pair sway-browser-sunshine --auto        # Pair as Moonlight client
ov sun status sway-browser-sunshine              # Verify health
# Sunshine Web UI at https://localhost:47990
# Chrome DevTools at localhost:9222
```

## Key Layers

- `/ov-layers:sway-desktop-sunshine` -- full desktop composition with Sunshine
- `/ov-layers:cuda` -- GPU support (via nvidia base)

## Related Images

- `/ov-images:sway-browser-sunshine-steam` -- same desktop + Steam gaming
- `/ov-images:openclaw-ollama-sway-sunshine` -- full-stack AI + Sunshine desktop
- `/ov-images:openclaw-sway-browser` -- similar but with VNC (wayvnc)

## When to Use This Skill

Use when the user asks about the sway-browser-sunshine image, lightweight Sunshine testing, or GPU-accelerated desktop without OpenClaw/Ollama.
