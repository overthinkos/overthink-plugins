---
name: openclaw-full
description: |
  Headless OpenClaw with all tool layers. No desktop environment.
  Currently disabled. Enable in images.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the openclaw-full image.
---

# openclaw-full

Headless OpenClaw image with maximal tool/skill coverage. No desktop, no VNC.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | openclaw-full |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 18789 (gateway), 9222 (CDP) |
| Status | **disabled** (set `enabled: true` in images.yml) |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (external base)
2. `openclaw-full` metalayer (28 layers):
   - `openclaw` — AI gateway
   - `chrome` — headless browser + CDP
   - `claude-code` — Claude Code CLI
   - 25 tool layers (codex, gemini, clawhub, mcporter, oracle, xurl, summarize, playwright, blogwatcher, gifgrep, wacli, goplaces, songsee, sag, camsnap, gogcli, ordercli, himalaya, uv, nano-pdf, gh, tmux, ffmpeg, ripgrep, sqlite)

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway | HTTP |
| 9222 | Chrome DevTools | HTTP |

## Quick Start

```bash
# Enable in images.yml first (remove enabled: false)
ov build openclaw-full
ov enable openclaw-full
ov start openclaw-full
```

## Key Layers

- `/ov-layers:openclaw-full` — Metalayer composition (all tools)
- `/ov-layers:openclaw` — AI gateway service

## Related Images

- `/ov-images:openclaw` — minimal (gateway only)
- `/ov-images:openclaw-sway-browser` — full tools + Sway desktop + VNC
- `/ov-images:openclaw-full-sway` — full tools + Sway desktop (disabled)
- `/ov-images:openclaw-full-ml` — full tools + ML + Ollama (disabled)

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-full image or the headless maximal OpenClaw deployment.
