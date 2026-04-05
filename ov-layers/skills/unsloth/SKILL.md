---
name: unsloth
description: |
  Unsloth LLM fine-tuning library with vLLM integration. Tier 1 post-install layer —
  no pixi.toml, requires pixi env from a parent layer (python-ml, jupyter-colab-ml, unsloth-studio).
  Use when working with Unsloth, LLM fine-tuning, or vLLM wheel installation.
---

# unsloth -- LLM fine-tuning (Tier 1 post-install layer)

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | None (requires pixi env from parent) |
| Volumes | `models` -> `~/.cache/huggingface` |
| Aliases | `unsloth` -> `unsloth` |
| Install files | `layer.yml`, `user.yml` |

## Architecture: Tier 1 Post-Install Layer

This layer has **no pixi.toml** and **no depends**. It installs pip packages into the pixi environment established by a Tier 2 "environment-owner" parent layer. The user.yml runs AFTER the parent's pixi COPY in the final image build.

**Cannot be used standalone** — must be composed into an environment-owner layer via the `layers:` field.

## Environment Variables

| Variable | Value |
|----------|-------|
| `UNSLOTH_SKIP_LLAMA_CPP_INSTALL` | `1` |
| `HF_HOME` | `~/.cache/huggingface` |

## Post-pixi Installs (user.yml)

1. **vLLM cu130 nightly wheel** (`pip install --no-deps`) — installed here because vLLM wheel must run after pixi env exists
2. **unsloth + unsloth-zoo** (`pip install --no-deps`) — incompatible with transformers 5.x in pixi solve
3. **vLLM 0.14 compatibility patch** — fixes `create_lora_manager` signature in `unsloth_zoo`

## Used In Layers (via `layers:` field)

- `/ov-layers:jupyter-colab-ml` — `layers: [llama-cpp, unsloth]`
- `/ov-layers:unsloth-studio` — `layers: [llama-cpp, unsloth]`

## Related Layers

- `/ov-layers:llama-cpp` — llama.cpp binaries (often paired as sibling Tier 1 layer)
- `/ov-layers:unsloth-studio` — Studio web UI (Tier 2 parent, owns pixi.toml)
- `/ov-layers:jupyter-colab-ml` — ML Jupyter with MCP (Tier 2 parent, owns pixi.toml)
- `/ov-layers:python-ml` — Core ML environment (Tier 2, uses vllm pip install in its own user.yml instead)

## When to Use This Skill

Use when the user asks about:

- Unsloth fine-tuning setup or configuration
- LLM fine-tuning environments
- The unsloth Python library or unsloth-zoo
- vLLM wheel installation inside pixi environments
- HuggingFace model cache volume
- The `unsloth` host alias
- The two-tier layer architecture for ML layers
