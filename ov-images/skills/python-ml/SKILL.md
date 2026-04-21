---
name: python-ml
description: |
  GPU-accelerated Python ML environment with CUDA, PyTorch, and llama.cpp.
  No Jupyter server — use as a base for ML workloads or interactive shell.
  MUST be invoked before building, deploying, configuring, or troubleshooting the python-ml image.
---

# python-ml

GPU-accelerated Python environment with ML libraries — PyTorch, transformers, vLLM, llama.cpp.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | agent-forwarding, python-ml |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Layer Composition

The `python-ml` layer is a **Tier 2 environment-owner meta-layer** that:
1. Owns the pixi.toml (core ML Python environment)
2. Composes `llama-cpp` (Tier 1 sub-layer) via `layers: [llama-cpp]`
3. Installs vLLM 0.19 wheel via a cmd task (after pixi env is established)

Build order: pixi environment → llama-cpp (binaries) → vLLM 0.19 wheel

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` (transitive)
3. `python-ml` — ML pixi environment (Tier 2, owns pixi.toml)
4. `llama-cpp` — llama.cpp binaries (Tier 1, via `layers:` field)

## Quick Start

```bash
ov image build python-ml
ov shell python-ml
# python -c "import torch; print(torch.cuda.is_available())"
```

## Key Layers

- `/ov-layers:python-ml` — ML Python packages via pixi (Tier 2 meta-layer)
- `/ov-layers:llama-cpp` — llama.cpp binaries (sub-layer)
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:nvidia` — parent (GPU without ML packages)
- `/ov-images:jupyter-ml` — adds JupyterLab + collaboration + MCP + unsloth on top of ML stack
- `/ov-images:jupyter` — legacy Jupyter with ML stack (monolithic)
- `/ov-images:unsloth-studio` — fine-tuning UI with similar ML stack

## Verification

After `ov image build`:
- `ov shell python-ml -c "python -c 'import torch; print(torch.cuda.is_available())'"` — CUDA OK
- `ov shell python-ml -c "python -c 'import vllm; print(vllm.__version__)'"` — vLLM OK
- `ov shell python-ml -c "ls ~/llama.cpp/llama-quantize"` — llama.cpp OK

## When to Use This Skill

**MUST be invoked** when the task involves the python-ml image, ML training environments, or GPU-accelerated Python. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
