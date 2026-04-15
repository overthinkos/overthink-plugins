---
name: unsloth-studio
description: |
  Unsloth Studio fine-tuning web UI on ports 8888/8000 with vLLM inference.
  Tier 2 environment-owner meta-layer composing llama-cpp + unsloth, owns pixi.toml.
  Use when working with Unsloth Studio, the fine-tuning web UI, or the unsloth-studio image.
---

# unsloth-studio -- Fine-tuning Studio (Tier 2 meta-layer)

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda`, `supervisord` |
| Sub-layers | `llama-cpp`, `unsloth` |
| Ports | 8888 (Studio UI), 8000 (vLLM API) |
| Volumes | `workspace` -> `~/workspace` |
| Service | `unsloth-studio` (supervisord) |
| Install files | `layer.yml`, `pixi.toml` |

## Architecture: Tier 2 Environment-Owner Meta-Layer

This layer **owns the pixi.toml** for the fine-tuning environment and composes two Tier 1 layers via `layers: [llama-cpp, unsloth]`. Build order: pixi environment → llama-cpp (binaries) → unsloth (vLLM wheel + unsloth pip + patch) → supervisord config.

## Environment Variables

| Variable | Value |
|----------|-------|
| `NVIDIA_PYTHON_PROJECT` | `~/.pixi` |
| `LD_LIBRARY_PATH` | `/usr/lib64:$HOME/llama.cpp` |

Plus from sub-layers: `LLAMA_CPP_PATH`, `UNSLOTH_SKIP_LLAMA_CPP_INSTALL`, `HF_HOME`

## Packages (pixi.toml)

Fine-tuning focused ML stack: PyTorch (CUDA 13.0), xformers, transformers, accelerate, vLLM runtime deps, HuggingFace (datasets, tokenizers, sentencepiece), fine-tuning (peft, trl, bitsandbytes, liger-kernel), GGUF tools

## Service

Runs `pixi run start-studio` which executes `unsloth studio -H 0.0.0.0 -p 8888`. The Studio launches its own vLLM API server on port 8000 for inference and synthetic data generation.

## Used In Images

- `/ov-images:unsloth-studio`

## Related Layers

- `/ov-layers:llama-cpp` — Sub-layer: llama.cpp binaries
- `/ov-layers:unsloth` — Sub-layer: vLLM + unsloth pip install + patch
- `/ov-layers:supervisord` — Process manager dependency
- `/ov-layers:jupyter-ml` — Alternative: ML Jupyter with MCP (same Tier 1 sub-layers)
- `/ov-layers:python-ml` — Alternative: core ML without UI

## When to Use This Skill

Use when the user asks about:

- Unsloth Studio web UI setup
- The unsloth-studio service or container
- Port 8888 or 8000 in the context of fine-tuning
- The two-tier layer architecture for ML layers
- Fine-tuning pixi environment ownership
