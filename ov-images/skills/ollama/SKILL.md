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
| Layers | agent-forwarding, ollama |
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
ov config ollama
ov start ollama
ov shell ollama -c "ollama pull llama3"
ov shell ollama -c "ollama run llama3 'Hello'"
```

## Host Alias

```bash
ov alias install ollama
# Now: ollama pull llama3  (runs inside the container)
```

## Service Environment

When deployed via `ov config ollama`, this image automatically provides `OLLAMA_HOST=http://ov-ollama:11434` to all other deployed containers via the `env_provides` mechanism. Use `--update-all` to propagate to already-deployed services:

```bash
ov config ollama --update-all
```

This means containers like `jupyter-colab-ml-notebook` automatically discover the Ollama endpoint without manual `OLLAMA_HOST` configuration.

## Key Layers

- `/ov-layers:ollama` — Ollama binary, supervisord service, model volume
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:nvidia` — parent (GPU without Ollama)
- `/ov-images:openclaw-ollama` — OpenClaw gateway + Ollama
- `/ov-images:openclaw-ollama-sway-browser` — full stack with desktop
- `/ov-images:jupyter-colab-ml-notebook` — Jupyter with Ollama integration notebooks (receives `OLLAMA_HOST` automatically via env_provides when ollama is deployed)

## Related Layers

- `/ov-layers:ollama` — the Ollama binary layer
- `/ov-layers:notebook-ollama` — 6 Jupyter notebooks demonstrating Ollama APIs (requests, OpenAI, ollama lib, Anthropic, HuggingFace, GPU)

## Verification

After `ov start`:
- `ov status ollama` — container running
- `ov service status ollama` — all services RUNNING
- `curl -s http://localhost:11434/api/tags` — Ollama API responds

## When to Use This Skill

**MUST be invoked** when the task involves the ollama image, LLM model serving, or the standalone Ollama deployment. Invoke this skill BEFORE reading source code or launching Explore agents.
