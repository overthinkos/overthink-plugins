---
name: jupyter
description: |
  Jupyter notebook server on port 8888 with CUDA GPU support and ML libraries.
  Use when working with Jupyter, notebooks, ML training, or llama.cpp.
---

# jupyter -- GPU-accelerated Jupyter notebook server

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda`, `supervisord` |
| Ports | 8888 |
| Service | `jupyter` (supervisord) |
| Install files | `user.yml`, `pixi.toml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NVIDIA_PYTHON_PROJECT` | `~/.pixi` |
| `LLAMA_CPP_PATH` | `~/llama.cpp` |
| `UNSLOTH_SKIP_LLAMA_CPP_INSTALL` | `1` |
| `LD_LIBRARY_PATH` | `/usr/lib64:$HOME/llama.cpp` |

## Usage

```yaml
# images.yml
jupyter:
  layers:
    - jupyter
```

## Used In Images

- `/ov-images:jupyter`

## Related Layers

- `/ov-layers:cuda` -- CUDA toolkit dependency
- `/ov-layers:supervisord` -- process manager dependency

## When to Use This Skill

Use when the user asks about:

- Jupyter notebook setup or configuration
- ML training environments
- llama.cpp integration
- Port 8888 service
- GPU-accelerated Python notebooks
