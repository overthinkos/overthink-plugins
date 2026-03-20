---
name: ollama
description: |
  Standalone Ollama LLM inference server with CUDA GPU support.
  Runs as a supervisord service on port 11434 with persistent model storage.
  MUST be invoked before building, deploying, configuring, or troubleshooting the ollama image.
---

# ollama

GPU-accelerated Ollama LLM inference server.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | ollama |
| Platforms | linux/amd64 |
| Ports | 11434 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` → `supervisord` (transitive)
3. `ollama` — LLM server, models volume

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 11434 | Ollama API | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| models | ~/.ollama | Model storage |

## Quick Start

```bash
ov build ollama
ov enable ollama
ov start ollama
ov shell ollama -c "ollama pull llama3"
ov shell ollama -c "ollama run llama3 'Hello'"
```

## Host Alias

```bash
ov alias install ollama
# Now: ollama pull llama3  (runs inside the container)
```

## Key Layers

- `/ov-layers:ollama` — Ollama binary, supervisord service, model volume
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:nvidia` — parent (GPU without Ollama)
- `/ov-images:openclaw-ollama` — OpenClaw gateway + Ollama
- `/ov-images:openclaw-ollama-sway-browser` — full stack with desktop

## Verification

After `ov start`:
- `ov status ollama` — container running
- `ov service status ollama` — all supervisord services RUNNING
- `curl -s http://localhost:11434/api/tags` — Ollama API responds

## When to Use This Skill

**MUST be invoked** when the task involves the ollama image, LLM model serving, or the standalone Ollama deployment. Invoke this skill BEFORE reading source code or launching Explore agents.
