---
name: jupyter-colab-ml-notebook
description: |
  Full CUDA ML JupyterLab image with finetuning, Ollama, and LLM course notebooks, CRDT MCP server, and real-time collaboration.
  Base: nvidia. Port 8888. Combines jupyter-colab-ml with 37 Unsloth fine-tuning notebooks, 6 Ollama integration notebooks, and 15 LLM course notebooks.
  MUST be invoked before building, deploying, or troubleshooting the jupyter-colab-ml-notebook image.
---

# jupyter-colab-ml-notebook -- GPU ML Jupyter with Fine-tuning Notebooks

## Image Definition

```yaml
jupyter-colab-ml-notebook:
  base: nvidia
  layers:
    - agent-forwarding
    - jupyter-colab-ml
    - notebook-templates
    - notebook-finetuning
    - notebook-ollama
    - notebook-llm-on-supercomputers
    - dbus
    - ov
  ports:
    - "8888:8888"
  platforms:
    - linux/amd64
```

## What's Different from jupyter-colab-ml

This image is identical to `jupyter-colab-ml` with two data layer additions:
- `notebook-finetuning` — seeds 37 Unsloth fine-tuning notebooks into `~/workspace/finetuning/`
- `notebook-ollama` — seeds 6 Ollama integration notebooks into `~/workspace/ollama/`

The Ollama notebooks require a running `ov-ollama` container on the same `ov` Podman network for API connectivity.

## Layer Composition

- `jupyter-colab-ml` — Tier 2 environment-owning meta-layer (PyTorch >= 2.10.0, vLLM 0.19, unsloth, LangChain, CRDT MCP via jupyter-colab-mcp sub-layer)
- `notebook-templates` — Starter notebooks (data layer, seeds ~/workspace)
- `notebook-finetuning` — 37 Unsloth fine-tuning notebooks (data layer, seeds ~/workspace/finetuning/)
- `notebook-ollama` — 6 Ollama integration notebooks (data layer, seeds ~/workspace/ollama/)
- `notebook-llm-on-supercomputers` — 15 LLM course notebooks (data layer, seeds ~/workspace/llms_on_supercomputers/)
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
| notebook-finetuning | workspace | finetuning/ | 37 Unsloth notebooks (SFT, GRPO, DPO, RLOO, QLoRA) |
| notebook-ollama | workspace | ollama/ | 6 Ollama API notebooks (requests, OpenAI, ollama lib, GPU, HuggingFace, Anthropic) |
| notebook-llm-on-supercomputers | workspace | llms_on_supercomputers/ | 15 LLM course notebooks (prompt engineering, RAG, fine-tuning) + datasets |

## File Layout in JupyterLab

```
~/workspace/
  getting-started.ipynb                    (from notebook-templates)
  finetuning/                               (from notebook-finetuning)
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
  ollama/                                   (from notebook-ollama)
    notebooks.yaml
    00_Ollama_Requests.ipynb
    01_Ollama_GPU.ipynb
    02_Ollama_OpenAI.ipynb
    03_Ollama_Library.ipynb
    04_Ollama_HuggingFace.ipynb
    05_Ollama_Anthropic.ipynb
  llms_on_supercomputers/                   (from notebook-llm-on-supercomputers)
    notebooks.yaml
    datasets/
    D0_00_Bazzite_AI_Setup.ipynb
    D1_01_Prompting_with_LangChain.ipynb
    D1_02_Prompt_templates_and_parsing.ipynb
    ... (15 notebooks total)
```

## Deploy

```bash
ov config jupyter-colab-ml-notebook
ov start jupyter-colab-ml-notebook
# JupyterLab: http://localhost:8888
# MCP endpoint: http://localhost:8888/mcp

# With bind-backed workspace (data seeded into local dir):
ov config jupyter-colab-ml-notebook --bind workspace=/path/to/project
```

## Notebook Workarounds

The notebook-finetuning include compatibility fixes for current library versions:
- **flex_attention disabled** in Ministral/Pixtral notebooks (transformers 5.5 bug)
- **max_memory={0: "14GB"}** for Pixtral-12B notebooks (accelerate device_map estimation fix)
- **packing=True** in all SFTConfig cells (TRL 1.0 requirement)
- **max_prompt_length removed** from DPO notebooks (deprecated in TRL 1.0)

## Verify

```bash
ov shell jupyter-colab-ml-notebook -c "pixi run verify-pytorch"
ov shell jupyter-colab-ml-notebook -c "pixi run verify-unsloth"
ov shell jupyter-colab-ml-notebook -c "pixi run verify-mcp"
ov shell jupyter-colab-ml-notebook -c "ls ~/workspace/finetuning/"
ov shell jupyter-colab-ml-notebook -c "ls ~/workspace/ollama/"
ov shell jupyter-colab-ml-notebook -c "ls ~/workspace/llms_on_supercomputers/"
```

## Related Images

- `/ov-images:jupyter-colab-ml` — Same stack without finetuning notebooks
- `/ov-images:jupyter-colab` — Lightweight variant (no CUDA, multi-arch)
- `/ov-images:unsloth-studio` — Unsloth Studio UI (different pixi env, same finetuning notebooks)

## When to Use This Skill

MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter-colab-ml-notebook image.
