---
name: python-ml
description: |
  Core ML/AI Python environment with PyTorch, vLLM runtime deps, and CUDA support.
  Tier 2 environment-owner meta-layer that composes llama-cpp.
  Use when working with machine learning, PyTorch, HuggingFace, or GPU computing.
---

# python-ml -- Core ML Python environment (Tier 2 meta-layer)

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda` |
| Sub-layers | `llama-cpp` |
| Install files | `layer.yml`, `pixi.toml`, `user.yml` |

## Architecture: Tier 2 Environment-Owner Meta-Layer

This layer **owns the pixi.toml** for the core ML Python environment and composes the `llama-cpp` Tier 1 layer via `layers: [llama-cpp]`. Build order: pixi environment → llama-cpp (binaries) → python-ml user.yml (vLLM wheel).

## Environment Variables

| Variable | Value |
|----------|-------|
| `NVIDIA_PYTHON_PROJECT` | `~/.pixi` |
| `LD_LIBRARY_PATH` | `/usr/lib64:$HOME/llama.cpp` |

Plus from `llama-cpp` sub-layer:

| Variable | Value |
|----------|-------|
| `LLAMA_CPP_PATH` | `~/llama.cpp` |
| PATH (appended) | `~/llama.cpp` |

## Packages (pixi.toml)

**PyPI:** PyTorch >= 2.9.1 (CUDA 13.0), xformers, transformers, accelerate, safetensors, numpy, scipy, einops, pillow, kornia, spandrel, torchsde, vLLM runtime deps (blake3, flashinfer, numba, ray, xgrammar, etc.), gguf, pydantic, aiohttp

## Post-pixi Installs (user.yml)

- **vLLM cu130 nightly wheel** (`pip install --no-deps`)

## Used In Images

- `/ov-images:python-ml`
- `/ov-images:immich-ml`

## Related Layers

- `/ov-layers:llama-cpp` — Sub-layer: llama.cpp binaries (composed via `layers:`)
- `/ov-layers:cuda` — CUDA toolkit dependency
- `/ov-layers:jupyter-colab-ml` — Full ML + Jupyter variant (superset of python-ml's pixi env)
- `/ov-layers:unsloth-studio` — Fine-tuning variant (similar pixi env + unsloth)

## When to Use This Skill

Use when the user asks about:

- Machine learning Python environment
- PyTorch, transformers, or vLLM setup
- CUDA Python integration
- The `python-ml` layer, its packages, or its meta-layer composition
- The two-tier layer architecture for ML layers
