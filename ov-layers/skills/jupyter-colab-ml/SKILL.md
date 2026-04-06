---
name: jupyter-colab-ml
description: |
  Full CUDA ML stack + JupyterLab with real-time collaboration and CRDT MCP server on port 8888.
  Use when working with GPU-accelerated Jupyter notebooks, ML training with collaboration,
  or the jupyter-colab-ml layer.
---

# jupyter-colab-ml -- Full ML + JupyterLab with CRDT MCP

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda`, `supervisord` |
| Sub-layers | `llama-cpp`, `unsloth`, `jupyter-colab-mcp` |
| Ports | 8888 |
| Service | `jupyter-colab-ml` (supervisord) |
| Volume | `workspace` at `~/workspace` |
| Install files | `layer.yml`, `pixi.toml`, `user.yml` |

## Architecture: Environment-Owning Meta-Layer

This is a **Tier 2 "environment owner"** layer that:
1. Owns the pixi.toml with ALL Python dependencies (Jupyter + ML + vLLM runtime deps)
2. Composes three Tier 1 sub-layers via `layers: [llama-cpp, unsloth, jupyter-colab-mcp]`
3. MCP extension installed by the `jupyter-colab-mcp` sub-layer (not directly in user.yml)

Build order: pixi environment → llama-cpp (binaries) → unsloth (vllm wheel + unsloth pip + patch) → jupyter-colab-mcp (MCP extension)

## Key Packages

**conda-forge:** JupyterLab >= 4.4.0, jupyter-resource-usage, jupyterlab-git, jupyterlab-lsp, jupyterlab-spellchecker, tensorboard, wandb, matplotlib, seaborn, pandas, numpy, scikit-learn, scipy, polars, pyarrow, dask, duckdb, altair, papermill, marimo, mkdocs, black, pytest

**PyPI (ML Core):** PyTorch >= 2.10.0 (CUDA 13.0), xformers, transformers >= 5.0.0rc1, accelerate, einops, kornia, spandrel, torchsde

**PyPI (vLLM Runtime):** blake3, flashinfer-python, numba, ray, xgrammar, and 25+ more runtime deps

**PyPI (Fine-tuning):** peft, trl, bitsandbytes, deepspeed, liger-kernel

**PyPI (LangChain):** langchain, langchain-core, langchain-openai, langchain-community, langchain-classic, langchain-anthropic, langchain-huggingface, langchain-ollama, chromadb, faiss-cpu

**PyPI (Evaluation):** evidently (with llm extras), evaluate, sacrebleu, rouge-score, nltk, bertviz

**PyPI (APIs):** openai, anthropic, gradio, ollama (client)

**PyPI (Collaboration):** jupyter-collaboration >= 4.1.0

**RPM:** git, gcc, gcc-c++

## Environment

| Variable | Value | Purpose |
|----------|-------|---------|
| `NVIDIA_PYTHON_PROJECT` | `~/.pixi` | NVIDIA driver → pixi env mapping |
| `LD_LIBRARY_PATH` | `/usr/lib64:$HOME/llama.cpp` | CUDA libs + llama.cpp shared libs |
| `LLAMA_CPP_PATH` | `~/llama.cpp` | (from llama-cpp sub-layer) |
| `UNSLOTH_SKIP_LLAMA_CPP_INSTALL` | `1` | (from unsloth sub-layer) |
| `HF_HOME` | `~/.cache/huggingface` | (from unsloth sub-layer) |

## MCP Server Extension

Same CRDT MCP server as `/ov-layers:jupyter-colab` — 13 tools for programmatic notebook access (list, get, create, update, insert, delete, execute cells, watch changes, collaboration awareness). See `/ov-layers:jupyter-colab` for full tool reference.

Endpoint: `http://localhost:8888/mcp` (Streamable HTTP, MCP spec 2025-11-25)

## Comparison

| | jupyter-colab | jupyter-colab-ml | jupyter |
|---|---|---|---|
| Base dep | supervisord | cuda, supervisord | cuda, supervisord |
| GPU | No | CUDA 13.0 | CUDA 13.0 |
| Platforms | amd64 + arm64 | amd64 only | amd64 only |
| MCP | CRDT (13 tools) | CRDT (13 tools) | jupyter-mcp-server |
| ML stack | No | Full (PyTorch, vLLM 0.19, unsloth) | Full (PyTorch, vLLM 0.19, unsloth) |
| Volume | workspace | workspace | — |

## Used In Images

- `/ov-images:jupyter-colab-ml`
- `/ov-images:jupyter-colab-ml-finetuning`

## Related Layers

- `/ov-layers:jupyter-colab` — Lightweight variant (no CUDA, multi-arch)
- `/ov-layers:jupyter` — Legacy monolithic variant
- `/ov-layers:llama-cpp` — Sub-layer: llama.cpp binaries
- `/ov-layers:unsloth` — Sub-layer: vLLM wheel + fine-tuning + vLLM patch
- `/ov-layers:jupyter-colab-mcp` — Sub-layer: CRDT MCP extension

## When to Use This Skill

Use when the user asks about:

- GPU-accelerated Jupyter with collaboration
- ML training notebooks with CRDT MCP
- The `jupyter-colab-ml` layer
- Combining jupyter-colab features with CUDA ML
