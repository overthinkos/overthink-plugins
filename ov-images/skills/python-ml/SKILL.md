---
name: python-ml
description: |
  GPU-accelerated Python ML environment with CUDA, PyTorch, and llama.cpp.
  No Jupyter server — use as a base for ML workloads or interactive shell.
  Use when working with the python-ml image or ML training environments.
---

# python-ml

GPU-accelerated Python environment with ML libraries — PyTorch, transformers, llama.cpp.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | python-ml |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` (transitive)
3. `python-ml` — ML pixi environment, llama.cpp build

## Quick Start

```bash
ov build python-ml
ov shell python-ml
# python -c "import torch; print(torch.cuda.is_available())"
```

## Key Layers

- `/ov-layers:python-ml` — ML Python packages via pixi, llama.cpp
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:nvidia` — parent (GPU without ML packages)
- `/ov-images:jupyter` — adds Jupyter notebook server on top of ML stack

## Verification

After `ov build`:
- `ov list` — image appears in list
- `ov shell python-ml` — interactive shell works
- `ov shell python-ml -c "python3 --version"` — Python available

## When to Use This Skill

Use when the user asks about the python-ml image, ML training environments, PyTorch in containers, or GPU-accelerated Python.
