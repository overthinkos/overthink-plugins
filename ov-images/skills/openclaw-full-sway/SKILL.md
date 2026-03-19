---
name: openclaw-full-sway
description: |
  Maximal OpenClaw deployment with Sway desktop, VNC, and all tools.
  Use when working with the openclaw-full-sway image or desktop OpenClaw deployments.
---

# openclaw-full-sway

Maximal OpenClaw with Sway Wayland desktop, VNC access, Chrome GUI, and all tool layers.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | openclaw-full, sway-desktop |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222 |
| Tunnel | tailscale (all ports) |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (Sway desktop) | TCP |
| 9222 | Chrome DevTools Protocol | HTTP (relay) |

## Quick Start

```bash
ov build openclaw-full-sway
ov start openclaw-full-sway
# Gateway at http://localhost:18789
# VNC desktop at localhost:5900
```

## Related Images

- `/ov-images:openclaw-full` -- same tools, no desktop
- `/ov-images:openclaw-full-ml` -- adds ML tools + ollama
- `/ov-images:openclaw-sway-browser` -- lighter variant with fewer tools

## When to Use This Skill

Use when the user asks about the desktop openclaw-full image, VNC-accessible OpenClaw, or comparing OpenClaw desktop variants.
