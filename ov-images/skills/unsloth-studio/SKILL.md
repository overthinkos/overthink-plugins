---
name: unsloth-studio
description: |
  Unsloth Studio fine-tuning web UI with CUDA GPU support, vLLM inference, and llama.cpp.
  Runs as a supervisord service on ports 8888 (Studio) and 8000 (vLLM API).
  MUST be invoked before building, deploying, configuring, or troubleshooting the unsloth-studio image.
---

# unsloth-studio

Unsloth Studio web UI for LLM fine-tuning with GPU acceleration.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | agent-forwarding, unsloth-studio, notebook-finetuning, dbus, ov |
| Platforms | linux/amd64 |
| Ports | 8888, 8000 |
| Registry | ghcr.io/overthinkos |

## Layer Composition

The `unsloth-studio` layer is a **Tier 2 environment-owner meta-layer** that:
1. Owns the pixi.toml (fine-tuning Python environment)
2. Composes two Tier 1 sub-layers via `layers: [llama-cpp, unsloth]`
3. Defines the supervisord service for the Studio web UI

Build order: pixi environment → llama-cpp (binaries) → unsloth (vLLM 0.19 wheel + unsloth pip + torch.compile patch) → supervisord config

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` → `supervisord` (transitive)
3. `unsloth-studio` — Tier 2 meta-layer (owns pixi.toml, service config)
4. `llama-cpp` — llama.cpp binaries (Tier 1, via `layers:`)
5. `unsloth` — vLLM 0.19 + unsloth pip install + torch.compile patch (Tier 1, via `layers:`)

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8888 | Unsloth Studio UI | HTTP |
| 8000 | vLLM API server | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| models | ~/.cache/huggingface | HuggingFace model cache |
| workspace | ~/workspace | Training data and outputs |

## Quick Start

```bash
ov image build unsloth-studio
ov config unsloth-studio
ov start unsloth-studio
# Open http://localhost:8888
```

## Key Layers

- `/ov-layers:unsloth-studio` — Studio web UI service + pixi.toml (Tier 2)
- `/ov-layers:llama-cpp` — llama.cpp binaries (Tier 1 sub-layer)
- `/ov-layers:unsloth` — vLLM 0.19 + unsloth fine-tuning + torch.compile patch (Tier 1 sub-layer)
- `/ov-layers:notebook-finetuning` — 37 Unsloth fine-tuning notebooks provisioned into workspace volume
- `/ov-layers:nvidia` — GPU runtime and CDI device auto-detection (base)
- `/ov-layers:cuda` — CUDA toolkit and libraries (via nvidia base)
- `/ov-layers:dbus` — session bus for desktop notifications
- `/ov-layers:ov` — in-container `ov` binary (enables `ov test dbus notify`)
- `/ov-layers:agent-forwarding` — SSH/GPG/direnv agent forwarding

## Related Images

- `/ov-images:nvidia` — parent (GPU without Studio)
- `/ov-images:jupyter-ml` — alternative ML UI with JupyterLab + CRDT MCP (same Tier 1 sub-layers)
- `/ov-images:python-ml` — ML libraries without any UI
- `/ov-images:jupyter` — legacy Jupyter with ML (shares port 8888)

## Verification

After `ov start`:
- `ov status unsloth-studio` — container running
- `ov service status unsloth-studio` — all services RUNNING
- `curl -s -o /dev/null -w '%{http_code}' http://localhost:8888` — Studio HTTP returns 200

## When to Use This Skill

**MUST be invoked** when the task involves the unsloth-studio image, LLM fine-tuning via web UI, or Unsloth Studio deployment. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
