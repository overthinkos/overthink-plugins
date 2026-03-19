---
name: openclaw
description: |
  Headless OpenClaw AI gateway image. Runs the gateway on port 18789
  without a desktop environment. Use when working with the headless
  openclaw deployment or comparing openclaw image variants.
---

# openclaw

Headless OpenClaw AI gateway — no desktop, no browser, just the gateway service.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | openclaw |
| Platforms | linux/amd64 |
| Ports | 18789 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `pixi` → `python` → `supervisord` (transitive)
3. `nodejs` (transitive via openclaw)
4. `openclaw` — gateway on :18789, data volume

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |

## Quick Start

```bash
ov build openclaw
ov start openclaw
# Gateway at http://localhost:18789
```

## Key Layers

- `/overthink-layers:openclaw` — gateway npm package, supervisord service, data volume

## Related Images

- `/overthink-images:openclaw-sway-browser` — adds desktop + Chrome for browser automation
- `/overthink-images:openclaw-ollama` — adds local LLM inference
- `/overthink-images:openclaw-ollama-sway-browser` — full stack with desktop + LLM

## When to Use This Skill

Use when the user asks about the headless openclaw image, deploying openclaw without a desktop, or comparing openclaw image variants.
