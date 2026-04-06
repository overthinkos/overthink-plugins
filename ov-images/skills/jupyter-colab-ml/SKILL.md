---
name: jupyter-colab-ml
description: |
  Full CUDA ML JupyterLab image with real-time collaboration and CRDT MCP server.
  Base: nvidia. Port 8888. GPU-accelerated ML training + collaborative notebooks.
  MUST be invoked before building, deploying, or troubleshooting the jupyter-colab-ml image.
---

# jupyter-colab-ml -- GPU ML Jupyter with CRDT MCP

## Image Definition

```yaml
jupyter-colab-ml:
  base: nvidia
  layers:
    - agent-forwarding
    - jupyter-colab-ml
    - notebook-templates
    - dbus
    - ov
  ports:
    - "8888:8888"
  platforms:
    - linux/amd64
```

## Layer Composition

The `jupyter-colab-ml` layer is a Tier 2 environment-owning meta-layer that composes:
- `llama-cpp` — llama.cpp prebuilt binaries + GGUF tools
- `unsloth` — vLLM 0.19 cu130 inference engine + fine-tuning (unsloth + unsloth-zoo) + vLLM torch.compile patch
- `jupyter-colab-mcp` — CRDT MCP extension (fastmcp + jupyter_colab_mcp package)

Additional layers from the image:
- `agent-forwarding` — SSH/GPG agent forwarding
- `notebook-templates` — Starter notebooks (data layer, seeds ~/workspace)
- `dbus` — D-Bus session bus
- `ov` — Overthink CLI

## Base Chain

```
quay.io/fedora/fedora:43
└── fedora
    └── fedora-nonfree (RPM Fusion repos)
        └─�� nvidia (NVIDIA drivers + container toolkit)
            └── jupyter-colab-ml
```

## Ports

| Port | Service |
|------|---------|
| 8888 | JupyterLab + MCP endpoint at `/mcp` |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| workspace | ~/workspace | Persistent notebook storage |
| models | ~/.cache/huggingface | HuggingFace model cache (from unsloth sub-layer) |

## Key Capabilities

- **JupyterLab** with real-time collaboration (jupyter-collaboration, Y-CRDT)
- **CRDT MCP Server** at `/mcp` — 13 tools for programmatic notebook access
- **PyTorch** >= 2.10.0 with CUDA 13.0
- **vLLM** 0.19 inference engine
- **Unsloth** fine-tuning (LoRA, QLoRA)
- **LangChain** RAG stack (chromadb, faiss-cpu)
- **Evaluation** tools (evidently, sacrebleu, rouge-score)
- **Notebook templates** seeded into ~/workspace

## Deploy

```bash
ov config jupyter-colab-ml
ov start jupyter-colab-ml
# JupyterLab: http://localhost:8888
# MCP endpoint: http://localhost:8888/mcp
```

## Verify

```bash
ov shell jupyter-colab-ml -c "pixi run verify-pytorch"
ov shell jupyter-colab-ml -c "pixi run verify-vllm"
ov shell jupyter-colab-ml -c "pixi run verify-unsloth"
ov shell jupyter-colab-ml -c "pixi run verify-mcp"
ov shell jupyter-colab-ml -c "pixi run verify-collaboration"
```

## Comparison with Other Jupyter Images

| | jupyter-colab | jupyter-colab-ml | jupyter |
|---|---|---|---|
| Base | fedora | nvidia | nvidia |
| CUDA | No | Yes | Yes |
| Arch | amd64 + arm64 | amd64 | amd64 |
| MCP | CRDT (13 tools) | CRDT (13 tools) | jupyter-mcp-server |
| Collaboration | Yes | Yes | Yes |
| ML Stack | No | Full | Full |
| Volume | workspace | workspace + models | — |
| Notebook dir | ~/workspace | ~/workspace | ~/workspace |

## Related Images

- `/ov-images:jupyter-colab-ml-finetuning` — Same stack with fine-tuning notebooks
- `/ov-images:jupyter-colab` — Lightweight variant (no CUDA, multi-arch)
- `/ov-images:jupyter` — Legacy monolithic variant
- `/ov-images:python-ml` — ML base without Jupyter

## When to Use This Skill

MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter-colab-ml image.
