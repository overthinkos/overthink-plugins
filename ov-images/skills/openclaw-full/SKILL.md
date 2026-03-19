---
name: openclaw-full
description: |
  Maximal headless OpenClaw deployment with all feasible tools and skills.
  Use when working with the openclaw-full image or comparing openclaw variants.
---

# openclaw-full

Maximal OpenClaw deployment — gateway + Chrome + all tool layers for maximum skill coverage. No desktop.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | openclaw-full (metalayer: 28 layers) |
| Platforms | linux/amd64 |
| Ports | 18789, 9222 |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 9222 | Chrome DevTools Protocol | HTTP (relay) |

## Quick Start

```bash
ov build openclaw-full
ov start openclaw-full
# Gateway at http://localhost:18789
# Chrome CDP at http://localhost:9222
```

## Included Tools

npm: codex, gemini, clawhub, mcporter, oracle, xurl, summarize, playwright, claude-code
Go: blogwatcher, gifgrep, wacli, goplaces, songsee, sag, camsnap, gogcli, ordercli
Cargo: himalaya
Python: uv, nano-pdf
RPM: gh, git, tmux, ffmpeg, ripgrep, sqlite

## Related Images

- `/ov-images:openclaw` -- minimal gateway only
- `/ov-images:openclaw-full-sway` -- adds desktop + VNC
- `/ov-images:openclaw-full-ml` -- adds ML tools (whisper, sherpa-onnx) + ollama

## When to Use This Skill

Use when the user asks about the headless openclaw-full image, maximal OpenClaw deployments, or which tools are included.
