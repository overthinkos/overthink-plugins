---
name: openclaw-full
description: |
  Headless OpenClaw with all tool layers. No desktop environment.
  Currently disabled. Enable in image.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the openclaw-full image.
---

# openclaw-full

Headless OpenClaw image with maximal tool/skill coverage. No desktop, no VNC.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, openclaw-full |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 18789 (gateway), 9222 (CDP) |
| Status | **disabled** (set `enabled: true` in image.yml) |
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
# Enable in image.yml first (remove enabled: false)
ov image build openclaw-full
ov config openclaw-full
ov start openclaw-full
```

## Key Layers

- `/ov-openclaw:openclaw-full` — Metalayer composition (all tools)
- `/ov-openclaw:openclaw` — AI gateway service

## Related Images

- `/ov-openclaw:openclaw` — minimal (gateway only)
- `/ov-openclaw:openclaw-sway-browser` — full tools + Sway desktop + VNC
- `/ov-openclaw:openclaw-full-sway` — full tools + Sway desktop (disabled)
- `/ov-openclaw:openclaw-full-ml` — full tools + ML + Ollama (disabled)

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-full image or the headless maximal OpenClaw deployment.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
