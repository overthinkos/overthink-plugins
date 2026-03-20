---
name: openclaw-sway-browser
description: |
  Maximal OpenClaw deployment with Sway desktop, Chrome, VNC, and all tool layers.
  Includes all feasible OpenClaw skill dependencies. Use when working with
  openclaw browser automation, desktop deployments, or full tool coverage.
---

# openclaw-sway-browser

Maximal OpenClaw gateway with full Wayland desktop, Chrome browser, VNC access, and all tool layers for maximum skill coverage.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | openclaw-full (metalayer: 28 layers), sway-desktop |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222 |
| Tunnel | tailscale (all ports) |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (wayvnc) | TCP |
| 9222 | Chrome DevTools | HTTP |

## Included Tools

npm: codex, gemini, clawhub, mcporter, oracle, xurl, summarize, playwright, claude-code
Go: blogwatcher, gifgrep, wacli, goplaces, songsee, sag, camsnap, gogcli, ordercli
Cargo: himalaya
Python: uv, nano-pdf
RPM: gh, git, tmux, ffmpeg, ripgrep, sqlite

## Quick Start

```bash
ov build openclaw-sway-browser
ov enable openclaw-sway-browser
ov start openclaw-sway-browser
# Gateway at http://localhost:18789
# VNC desktop at localhost:5900
```

## Key Layers

- `/ov-layers:openclaw-full` — metalayer composing openclaw + chrome + 26 tool layers
- `/ov-layers:sway-desktop` — full desktop (pipewire, wayvnc, chrome-sway, terminal, file manager, waybar)

## Related Images

- `/ov-images:openclaw` — headless gateway only (minimal)
- `/ov-images:openclaw-ollama-sway-browser` — adds CUDA, Ollama, Whisper, sherpa-onnx for ML

## Verification

After `ov start`:
- `ov status openclaw-sway-browser` — container running
- `ov service status openclaw-sway-browser` — all supervisord services RUNNING
- `curl -s http://localhost:18789` — OpenClaw gateway responds
- VNC client connects to `localhost:5900` — desktop accessible
- `curl -s http://localhost:9222/json/version` — Chrome DevTools responds

## Port Relay Architecture

OpenClaw (18789) and Chrome DevTools (9222) both use port relay (socat) — services bind to loopback, socat forwards from the container interface. This avoids origin/security checks that block non-loopback connections.

## When to Use This Skill

Use when the user asks about openclaw-sway-browser, OpenClaw with browser automation, desktop gateway deployments, VNC access, or which tools are included.
