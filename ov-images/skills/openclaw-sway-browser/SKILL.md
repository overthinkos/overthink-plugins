---
name: openclaw-sway-browser
description: |
  OpenClaw AI gateway with full Sway desktop and Chrome browser.
  Includes VNC, Chrome DevTools, and Tailscale tunnel support.
  Use when working with openclaw browser automation or desktop deployments.
---

# openclaw-sway-browser

OpenClaw gateway with full Wayland desktop, Chrome browser, and VNC access.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | openclaw, sway-desktop |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222 |
| Tunnel | tailscale (all ports) |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `pixi` → `python` → `supervisord` (transitive)
3. `nodejs` (transitive)
4. `openclaw` — gateway on :18789
5. `dbus` → `sway` (transitive via sway-desktop)
6. `pipewire` — audio server
7. `wayvnc` — VNC on :5900
8. `socat` → `chrome` → `chrome-sway` — Chrome on :9222
9. `xfce4-terminal`, `thunar`, `waybar` — desktop apps

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (wayvnc) | TCP |
| 9222 | Chrome DevTools | HTTP |

## Tunnel

Tailscale tunnel configured with `ports: all` — exposes all three ports via Tailscale serve.

## Quick Start

```bash
ov build openclaw-sway-browser
ov enable openclaw-sway-browser    # Deploy with quadlet + tunnel
ov start openclaw-sway-browser
```

## Key Layers

- `/ov-layers:openclaw` — AI gateway service
- `/ov-layers:sway-desktop` — full desktop composition (pipewire, wayvnc, chrome-sway, terminal, file manager, waybar)

## Related Images

- `/ov-images:openclaw` — headless (no desktop)
- `/ov-images:openclaw-ollama-sway-browser` — adds local LLM with Ollama

## When to Use This Skill

Use when the user asks about openclaw-sway-browser, OpenClaw with browser automation, desktop gateway deployments, or VNC access to OpenClaw.
