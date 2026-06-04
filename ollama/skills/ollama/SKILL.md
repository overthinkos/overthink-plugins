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

## GPU is image-level — the `ollama` layer is GPU-agnostic

The `ollama` **layer** installs the distro-agnostic Ollama binary (a tarball
extracted to `/usr`) and a supervisord `ollama serve` service. The binary
auto-detects the GPU at runtime and falls back to CPU inference when none is
present, so the layer carries **no `cuda` dependency** — it only `require:`s
`supervisord`. GPU support is a **composition choice made at the image level**:

- This `ollama` image keeps GPU acceleration via `base: nvidia` (the `nvidia`
  image composes the `cuda` layer, inherited through the base chain).
- CPU-only consumers compose the `ollama` layer on a non-NVIDIA base and get
  CPU inference for free — e.g. `/ov-openclaw:openclaw-desktop` (cachyos base,
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
ov image build ollama
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

When deployed via `ov config ollama`, this image automatically provides `OLLAMA_HOST=http://ov-ollama:11434` to all other deployed containers via the `env_provide` mechanism. Use `--update-all` to propagate to already-deployed services:

```bash
ov config ollama --update-all
```

This means containers like `jupyter-ml-notebook` automatically discover the Ollama endpoint without manual `OLLAMA_HOST` configuration.

## Key Layers

- `/ov-ollama:ollama` — Ollama binary, supervisord service, model volume
- `/ov-distros:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-distros:nvidia` — parent (GPU without Ollama)
- **CachyOS variant** — `cachyos.ollama` is the CachyOS GPU sibling (built on the `cachyos.nvidia` GPU base) in the `overthinkos/cachyos` submodule. See `/ov-distros:cachyos`.
- `/ov-openclaw:openclaw-desktop` — composes the `ollama` layer CPU-only (cachyos base, no `cuda`) alongside a streaming desktop + the openclaw gateway + the nested ov toolchain
- `/ov-jupyter:jupyter-ml-notebook` — Jupyter with Ollama integration notebooks (receives `OLLAMA_HOST` automatically via env_provide when ollama is deployed)
- `/ov-openwebui:openwebui` — Open WebUI (receives `OLLAMA_HOST` via env_provide, auto-configures as `OLLAMA_BASE_URL`)

## Related Layers

- `/ov-ollama:ollama` — the Ollama binary layer
- `/ov-jupyter:notebook-ollama` — 6 Jupyter notebooks demonstrating Ollama APIs (requests, OpenAI, ollama lib, Anthropic, HuggingFace, GPU)

## Verification

After `ov start`:
- `ov status ollama` — container running
- `ov service status ollama` — all services RUNNING
- `curl -s http://localhost:11434/api/tags` — Ollama API responds

## When to Use This Skill

**MUST be invoked** when the task involves the ollama image, LLM model serving, or the standalone Ollama deployment. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
