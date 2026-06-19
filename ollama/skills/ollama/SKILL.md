---
name: ollama
description: |
  Standalone Ollama LLM inference server with CUDA GPU support.
  Runs as a supervisord service on port 11434 with persistent model storage.
  MUST be invoked before building, deploying, configuring, or troubleshooting the ollama box.
---

# ollama

GPU-accelerated Ollama LLM inference server.

## Box Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Candies | agent-forwarding, ollama |
| Platforms | linux/amd64 |
| Ports | 11434 |
| Registry | ghcr.io/overthinkos |

## Full Candy Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` → `supervisord` (transitive)
3. `ollama` — LLM server, models volume

## GPU is box-level — the `ollama` candy is GPU-agnostic

The `ollama` **candy** installs the distro-agnostic Ollama binary (a tarball
extracted to `/usr`) and a supervisord `ollama serve` service. The binary
auto-detects the GPU at runtime and falls back to CPU inference when none is
present, so the candy carries **no `cuda` dependency** — it only `require:`s
`supervisord`. GPU support is a **composition choice made at the box level**:

- This `ollama` box keeps GPU acceleration via `base: nvidia` (the `nvidia`
  box composes the `cuda` candy, inherited through the base chain).
- CPU-only consumers compose the `ollama` candy on a non-NVIDIA base and get
  CPU inference for free — e.g. `/charly-openclaw:openclaw-desktop` (cachyos base,
  no `cuda`).

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
charly box build ollama
charly config ollama
charly start ollama
charly shell ollama -c "ollama pull llama3"
charly shell ollama -c "ollama run llama3 'Hello'"
```

## Host Alias

```bash
charly alias install ollama
# Now: ollama pull llama3  (runs inside the container)
```

## Service Environment

When deployed via `charly config ollama`, this box automatically provides `OLLAMA_HOST=http://charly-ollama:11434` to all other deployed containers via the `env_provide` mechanism. Use `--update-all` to propagate to already-deployed services:

```bash
charly config ollama --update-all
```

This means containers like `jupyter-ml-notebook` automatically discover the Ollama endpoint without manual `OLLAMA_HOST` configuration.

## Key Candies

- `/charly-ollama:ollama` — Ollama binary, supervisord service, model volume
- `/charly-distros:cuda` — GPU support (via nvidia base)

## Related Boxes

- `/charly-distros:nvidia` — parent (GPU without Ollama)
- **CachyOS variant** — `cachyos.ollama` is the CachyOS GPU sibling (built on the `cachyos.nvidia` GPU base) in the `overthinkos/cachyos` submodule. See `/charly-distros:cachyos`.
- `/charly-openclaw:openclaw-desktop` — composes the `ollama` candy CPU-only (cachyos base, no `cuda`) alongside a streaming desktop + the openclaw gateway + the nested charly toolchain
- `/charly-jupyter:jupyter-ml-notebook` — Jupyter with Ollama integration notebooks (receives `OLLAMA_HOST` automatically via env_provide when ollama is deployed)
- `/charly-openwebui:openwebui` — Open WebUI (receives `OLLAMA_HOST` via env_provide, auto-configures as `OLLAMA_BASE_URL`)

## Related Candies

- `/charly-ollama:ollama` — the Ollama binary candy
- `/charly-jupyter:notebook-ollama` — 6 Jupyter notebooks demonstrating Ollama APIs (requests, OpenAI, ollama lib, Anthropic, HuggingFace, GPU)

## Verification

After `charly start`:
- `charly status ollama` — container running
- `charly service status ollama` — all services RUNNING
- `curl -s http://localhost:11434/api/tags` — Ollama API responds

## When to Use This Skill

**MUST be invoked** when the task involves the ollama box, LLM model serving, or the standalone Ollama deployment. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/charly-image:image` — image family umbrella (`box:` entries in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — the embedded build vocabulary (distros, builders, init-systems)
