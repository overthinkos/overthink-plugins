---
name: jupyter-colab-ml-finetuning
description: |
  Full CUDA ML JupyterLab image with finetuning and Ollama notebooks, CRDT MCP server, and real-time collaboration.
  Base: nvidia. Port 8888. Combines jupyter-colab-ml with 37 Unsloth fine-tuning notebooks and 6 Ollama integration notebooks.
  MUST be invoked before building, deploying, or troubleshooting the jupyter-colab-ml-finetuning image.
---

# jupyter-colab-ml-finetuning -- GPU ML Jupyter with Fine-tuning Notebooks

## Image Definition

```yaml
jupyter-colab-ml-finetuning:
  base: nvidia
  layers:
    - agent-forwarding
    - jupyter-colab-ml
    - notebook-templates
    - finetuning-notebooks
    - ollama-notebooks
    - dbus
    - ov
  ports:
    - "8888:8888"
  platforms:
    - linux/amd64
```

## What's Different from jupyter-colab-ml

This image is identical to `jupyter-colab-ml` with two data layer additions:
- `finetuning-notebooks` — seeds 37 Unsloth fine-tuning notebooks into `~/workspace/finetuning/`
- `ollama-notebooks` — seeds 6 Ollama integration notebooks into `~/workspace/ollama/`

The Ollama notebooks require a running `ov-ollama` container on the same `ov` Podman network for API connectivity.

## Layer Composition

- `jupyter-colab-ml` — Tier 2 environment-owning meta-layer (PyTorch >= 2.10.0, vLLM 0.19, unsloth, LangChain, CRDT MCP via jupyter-colab-mcp sub-layer)
- `notebook-templates` — Starter notebooks (data layer, seeds ~/workspace)
- `finetuning-notebooks` — 37 Unsloth fine-tuning notebooks (data layer, seeds ~/workspace/finetuning/)
- `ollama-notebooks` — 6 Ollama integration notebooks (data layer, seeds ~/workspace/ollama/)
- `agent-forwarding` — SSH/GPG agent forwarding
- `dbus` — D-Bus session bus
- `ov` — Overthink CLI

## Ports

| Port | Service |
|------|---------|
| 8888 | JupyterLab + MCP endpoint at `/mcp` |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| workspace | ~/workspace | Persistent notebook storage |
| models | ~/.cache/huggingface | HuggingFace model cache (from unsloth sub-layer) |

## Data Layers

| Layer | Target | Dest | Contents |
|-------|--------|------|----------|
| notebook-templates | workspace | *(root)* | getting-started.ipynb |
| finetuning-notebooks | workspace | finetuning/ | 37 Unsloth notebooks (SFT, GRPO, DPO, RLOO, QLoRA) |
| ollama-notebooks | workspace | ollama/ | 6 Ollama API notebooks (requests, OpenAI, ollama lib, GPU, HuggingFace, Anthropic) |

## File Layout in JupyterLab

```
~/workspace/
  getting-started.ipynb                    (from notebook-templates)
  finetuning/                               (from finetuning-notebooks)
    .env.example
    notebooks.yaml
    00_Unsloth_Setup.ipynb
    01_FastInference_Llama.ipynb
    01_FastInference_Qwen.ipynb
    02_Vision_Training_Ministral.ipynb
    03_SFT_Training_Qwen.ipynb
    04_GRPO_Training_Qwen.ipynb
    05_DPO_Training_Qwen.ipynb
    06_Reward_Training_Qwen.ipynb
    07_RLOO_Training_Qwen.ipynb
    08_QLoRA_Alpha_Scaling_Ministral.ipynb
    ... (37 notebooks total)
  ollama/                                   (from ollama-notebooks)
    notebooks.yaml
    ollama_requests.ipynb
    ollama_openai.ipynb
    ollama_ollama.ipynb
    ollama_gpu.ipynb
    ollama_huggingface.ipynb
    ollama_anthropic.ipynb
```

## Deploy

```bash
ov config jupyter-colab-ml-finetuning
ov start jupyter-colab-ml-finetuning
# JupyterLab: http://localhost:8888
# MCP endpoint: http://localhost:8888/mcp

# With bind-backed workspace (data seeded into local dir):
ov config jupyter-colab-ml-finetuning --bind workspace=/path/to/project
```

## Notebook Workarounds

The finetuning-notebooks include compatibility fixes for current library versions:
- **flex_attention disabled** in Ministral/Pixtral notebooks (transformers 5.5 bug)
- **max_memory={0: "14GB"}** for Pixtral-12B notebooks (accelerate device_map estimation fix)
- **packing=True** in all SFTConfig cells (TRL 1.0 requirement)
- **max_prompt_length removed** from DPO notebooks (deprecated in TRL 1.0)

## Verify

```bash
ov shell jupyter-colab-ml-finetuning -c "pixi run verify-pytorch"
ov shell jupyter-colab-ml-finetuning -c "pixi run verify-unsloth"
ov shell jupyter-colab-ml-finetuning -c "pixi run verify-mcp"
ov shell jupyter-colab-ml-finetuning -c "ls ~/workspace/finetuning/"
ov shell jupyter-colab-ml-finetuning -c "ls ~/workspace/ollama/"
```

## Related Images

- `/ov-images:jupyter-colab-ml` — Same stack without finetuning notebooks
- `/ov-images:jupyter-colab` — Lightweight variant (no CUDA, multi-arch)
- `/ov-images:unsloth-studio` — Unsloth Studio UI (different pixi env, same finetuning notebooks)

## When to Use This Skill

MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter-colab-ml-finetuning image.
