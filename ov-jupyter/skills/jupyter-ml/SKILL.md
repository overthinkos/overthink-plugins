---
name: jupyter-ml
description: |
  Full CUDA ML JupyterLab image with real-time collaboration and CRDT MCP server.
  Base: nvidia. Port 8888. GPU-accelerated ML training + collaborative notebooks.
  MUST be invoked before building, deploying, or troubleshooting the jupyter-ml image.
---

# jupyter-ml -- GPU ML Jupyter with CRDT MCP

## Image Definition

```yaml
jupyter-ml:
  base: nvidia
  layers:
    - agent-forwarding
    - jupyter-ml
    - notebook-templates
    - dbus
    - ov
  ports:
    - "8888:8888"
  platforms:
    - linux/amd64
```

## Layer Composition

The `jupyter-ml` layer is a Tier 2 environment-owning meta-layer that composes:
- `llama-cpp` — llama.cpp prebuilt binaries + GGUF tools
- `unsloth` — vLLM 0.19 cu130 inference engine + fine-tuning (unsloth + unsloth-zoo) + vLLM torch.compile patch
- `jupyter-mcp` — CRDT MCP extension (fastmcp + jupyter_mcp package)

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
        └── nvidia (NVIDIA drivers + container toolkit)
            └── jupyter-ml
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

## Service Environment Integration

This image receives env_provides variables from infrastructure layers when they are deployed:

| Variable | Injected by | Value |
|----------|------------|-------|
| `OLLAMA_HOST` | `/ov-ollama:ollama` | `http://ov-ollama:11434` |
| `PGHOST` | `/ov-foundation:postgresql` | `ov-postgresql` |
| `PGPORT` | `/ov-foundation:postgresql` | `5432` |
| `REDIS_URL` | `/ov-foundation:redis` or `/ov-foundation:valkey` | `redis://ov-<image>:6379` |

These variables are injected automatically into the container environment at `ov config` time when the corresponding service is deployed. No manual `-e` flags needed.

## Key Capabilities

- **JupyterLab** with real-time collaboration (jupyter-collaboration, Y-CRDT)
- **CRDT MCP Server** at `/mcp` — 13 tools for programmatic notebook access
- **PyTorch** >= 2.10.0 with CUDA 13.0
- **vLLM** 0.19 inference engine
- **Unsloth** fine-tuning (LoRA, QLoRA)
- **LangChain** RAG stack (chromadb, faiss-cpu)
- **Evaluation** tools (evidently, sacrebleu, rouge-score)
- **Notebook templates** seeded into ~/workspace

## Quick Start

```bash
ov image build jupyter-ml
ov config jupyter-ml
ov start jupyter-ml
ov status jupyter-ml
ov logs jupyter-ml -f
# JupyterLab: http://localhost:8888
# MCP endpoint: http://localhost:8888/mcp
```

## Verify

```bash
ov shell jupyter-ml -c "pixi run verify-pytorch"
ov shell jupyter-ml -c "pixi run verify-vllm"
ov shell jupyter-ml -c "pixi run verify-unsloth"
ov shell jupyter-ml -c "pixi run verify-mcp"
ov shell jupyter-ml -c "pixi run verify-collaboration"
```

## Comparison with Other Jupyter Images

| | jupyter | jupyter-ml |
|---|---|---|
| Base | fedora | nvidia |
| CUDA | No | Yes |
| Arch | amd64 + arm64 | amd64 |
| MCP | CRDT (13 tools) | CRDT (13 tools) |
| Collaboration | Yes | Yes |
| ML Stack | No | Full |
| Volume | workspace | workspace + models |
| Notebook dir | ~/workspace | ~/workspace |

## Related Images

- `/ov-jupyter:jupyter-ml-notebook` — Same stack with fine-tuning notebooks
- `/ov-jupyter:jupyter` — Lightweight variant (no CUDA, multi-arch)
- `/ov-foundation:python-ml` — ML base without Jupyter

**MCP testing:** inherits 3 deploy-scope `mcp:` checks from the `jupyter-ml` layer (`ping`, `list-tools`, `call list_notebooks`). Run `ov test jupyter-ml --filter mcp` or probe ad-hoc with `ov eval mcp list-tools jupyter-ml`. See `/ov-build:mcp`.

## When to Use This Skill

MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter-ml image.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
