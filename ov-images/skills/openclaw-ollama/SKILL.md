---
name: openclaw-ollama
description: |
  Headless OpenClaw gateway with local Ollama LLM inference. GPU-accelerated,
  no desktop. Use when working with the headless openclaw+ollama deployment
  MUST be invoked before building, deploying, configuring, or troubleshooting the openclaw-ollama image.
---

# openclaw-ollama

Headless OpenClaw gateway with local LLM inference via Ollama — no desktop, no browser.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | openclaw, ollama |
| Platforms | linux/amd64 |
| Ports | 18789, 11434 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` → `supervisord` (transitive)
3. `nodejs` (transitive)
4. `openclaw` — gateway on :18789
5. `ollama` — LLM server on :11434

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 11434 | Ollama API | HTTP |

## Quick Start

```bash
ov build openclaw-ollama
ov enable openclaw-ollama
ov start openclaw-ollama
# Pull a model and configure OpenClaw to use it
ov shell openclaw-ollama -c "ollama pull llama3"
```

## Key Layers

- `/ov-layers:openclaw` — AI gateway service
- `/ov-layers:ollama` — LLM inference server

## Related Images

- `/ov-images:openclaw` — gateway only (no LLM, no GPU)
- `/ov-images:ollama` — LLM only (no gateway)
- `/ov-images:openclaw-ollama-sway-browser` — adds desktop + Chrome

## Verification

After `ov start`:
- `ov status openclaw-ollama` — container running
- `ov service status openclaw-ollama` — all supervisord services RUNNING
- `curl -s http://localhost:18789` — OpenClaw gateway responds
- `curl -s http://localhost:11434/api/tags` — Ollama API responds

## Port Relay Architecture

OpenClaw gateway (18789) uses port relay (socat) — the gateway binds to loopback, socat forwards from the container interface. This avoids the `allowedOrigins` requirement for the Control UI.

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-ollama image, running OpenClaw with local models, or headless openclaw+ollama deployment. Invoke this skill BEFORE reading source code or launching Explore agents.
