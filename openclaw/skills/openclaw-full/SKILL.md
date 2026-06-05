---
name: openclaw-full
description: |
  Headless OpenClaw with all tool layers. No desktop environment.
  MUST be invoked before building, deploying, or troubleshooting the openclaw-full image.
---

# openclaw-full

Headless OpenClaw image with maximal tool/skill coverage. No desktop, no VNC.

## Image Properties

| Property | Value |
|----------|-------|
| Base | cachyos |
| Layers | agent-forwarding, openclaw-full |
| Platforms | linux/amd64 |
| Ports | 18789 (gateway) |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `cachyos` (external base, Arch-derived — pacman/AUR)
2. `openclaw-full` metalayer (27 layers, no system browser):
   - `openclaw` — AI gateway
   - `claude-code` — Claude Code CLI
   - 25 tool layers (codex, gemini, clawhub, mcporter, oracle, xurl, summarize, playwright, blogwatcher, gifgrep, wacli, goplaces, songsee, sag, camsnap, gogcli, ordercli, himalaya, uv, nano-pdf, gh, tmux, ffmpeg, ripgrep, sqlite)

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway | HTTP |

## Quick Start

```bash
ov box build openclaw-full
ov config openclaw-full
ov start openclaw-full
```

## Key Layers

- `/ov-openclaw:openclaw-full` — Metalayer composition (all tools)
- `/ov-openclaw:openclaw` — AI gateway service

## Related Images

- `/ov-openclaw:openclaw` — minimal (gateway only)

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-full image or the headless maximal OpenClaw deployment.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
