---
name: unsloth
description: |
  Unsloth LLM fine-tuning library with CUDA GPU support, vLLM inference, and llama.cpp tools.
  Use when working with Unsloth, LLM fine-tuning, or the unsloth-studio layer.
---

# unsloth -- LLM fine-tuning environment

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda` |
| Volumes | `models` -> `~/.cache/huggingface` |
| Aliases | `unsloth` -> `unsloth` |
| Install files | `pixi.toml`, `user.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NVIDIA_PYTHON_PROJECT` | `~/.pixi` |
| `LLAMA_CPP_PATH` | `~/llama.cpp` |
| `UNSLOTH_SKIP_LLAMA_CPP_INSTALL` | `1` |
| `LD_LIBRARY_PATH` | `/usr/lib64:$HOME/llama.cpp` |
| `HF_HOME` | `~/.cache/huggingface` |

## Python Environment (pixi.toml)

Focused ML fine-tuning stack:
- PyTorch with CUDA 13.0 (cu130 wheels)
- xformers (memory-efficient attention)
- vLLM runtime dependencies (wheel installed --no-deps in user.yml)
- HuggingFace: transformers, accelerate, datasets, tokenizers, sentencepiece
- Fine-tuning: peft, trl, bitsandbytes, liger-kernel
- GGUF tools for model conversion

## Post-pixi Installs (user.yml)

- llama.cpp prebuilt binaries (quantize, cli, convert_hf_to_gguf.py)
- vLLM cu130 nightly wheel (--no-deps)
- unsloth + unsloth-zoo (--no-deps, incompatible with transformers 5.x in pixi solve)

## Usage

```yaml
# images.yml
my-finetuning-image:
  base: nvidia
  layers:
    - unsloth
```

## Used In Images

- `/ov-images:unsloth-studio` (via `unsloth-studio` metalayer)

## Related Layers

- `/ov-layers:cuda` -- CUDA toolkit dependency
- `/ov-layers:unsloth-studio` -- Studio web UI metalayer that composes this layer
- `/ov-layers:python-ml` -- Alternative ML environment (broader scope, no unsloth)
- `/ov-layers:jupyter` -- Jupyter notebooks with similar ML stack

## When to Use This Skill

Use when the user asks about:

- Unsloth fine-tuning setup or configuration
- LLM fine-tuning environments
- The unsloth Python library or unsloth-zoo
- HuggingFace model cache volume
- The `unsloth` host alias
