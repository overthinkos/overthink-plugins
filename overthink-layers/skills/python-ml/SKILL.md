---
name: python-ml
description: |
  ML/AI Python environment with PyTorch, transformers, vLLM, and CUDA support.
  Use when working with machine learning, PyTorch, HuggingFace, or GPU computing.
---

# python-ml -- ML/AI Python environment

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda` |
| Install files | `layer.yml`, `pixi.toml`, `user.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NVIDIA_PYTHON_PROJECT` | `~/.pixi` |
| `LLAMA_CPP_PATH` | `~/llama.cpp` |
| `UNSLOTH_SKIP_LLAMA_CPP_INSTALL` | `1` |
| `LD_LIBRARY_PATH` | `/usr/lib64:$HOME/llama.cpp` |

PATH additions: `~/llama.cpp`

## Packages

Pixi/PyPI: `torch` (CUDA 13.0), `torchvision`, `torchaudio`, `xformers`, `transformers`, `accelerate`, `safetensors`, `numpy`, `scipy`, `einops`, `pillow`, `kornia`, `spandrel`, `torchsde`, `vllm` (runtime deps), `gguf`, `pydantic`, `ray`, `aiohttp`, and more

Verification tasks: `pixi run verify-cuda`, `pixi run verify-env`, `pixi run verify-transformers`, `pixi run verify-vllm`

## Usage

```yaml
# images.yml
my-ml:
  base: nvidia
  layers:
    - python-ml
```

## Used In Images

- `/overthink-images:python-ml`
- `/overthink-images:immich-cuda`

## Related Layers

- `/overthink-layers:cuda` -- required CUDA toolkit dependency (not documented here)
- `/overthink-layers:pixi` -- transitive via python dependency chain

## When to Use This Skill

Use when the user asks about:

- Machine learning Python environment
- PyTorch, transformers, or vLLM setup
- CUDA Python integration
- ML model loading or inference
- The `python-ml` layer or its packages
