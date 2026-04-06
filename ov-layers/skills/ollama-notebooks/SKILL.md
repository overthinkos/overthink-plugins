---
name: ollama-notebooks
description: |
  Ollama integration notebook collection provisioned into the workspace volume at deploy time.
  6 Jupyter notebooks demonstrating Ollama via requests, OpenAI, ollama lib, Anthropic, HuggingFace, and GPU.
  Data-only layer — no packages, no services, no dependencies.
  Use when working with ollama-notebooks, Ollama API tutorials, or Jupyter+Ollama integration.
---

# ollama-notebooks -- Ollama integration notebook data layer

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none)* |
| Services | *(none)* |
| Volumes | `workspace` -> `~/workspace` (from jupyter-colab) |
| Data | `data/ollama` -> `workspace` volume, dest: `ollama` |
| Install files | *(none)* |

## How It Works

This is a **data layer** — it uses the `data:` field in `layer.yml` to map a directory of notebooks to a named volume with a subdirectory destination:

```yaml
info: "Ollama integration notebook collection"

data:
  - src: data/ollama
    volume: workspace
    dest: ollama
```

At build time, the contents of `data/ollama/` are staged into `/data/workspace/ollama/` inside the image.

At deploy time, `ov config` or `ov update` copies the staged data into the workspace volume at `<workspace>/ollama/`. The `dest: ollama` field places the notebooks in a subdirectory rather than the volume root.

## Included Notebooks

6 Jupyter notebooks covering different Ollama client libraries:

| Notebook | Library | Features |
|----------|---------|----------|
| `ollama_requests.ipynb` | `requests` | Raw REST API: list, show, generate, chat, stream, embed, copy, delete |
| `ollama_openai.ipynb` | `openai` | OpenAI-compatible API: completions, chat, multi-turn, stream, embed |
| `ollama_ollama.ipynb` | `ollama` | Native Python library: all API operations + model management |
| `ollama_gpu.ipynb` | `requests` | GPU verification: nvidia-smi, inference metrics, VRAM monitoring |
| `ollama_huggingface.ipynb` | `ollama` | HuggingFace GGUF model import, verification, inference |
| `ollama_anthropic.ipynb` | `anthropic` | Anthropic-compatible API: chat, system prompts, streaming, tool calling, vision |

**Manifest:** `notebooks.yaml` — structured catalog with title, description, and ordering.

## Network Connectivity

The notebooks connect to Ollama at `http://ov-ollama:11434` via Podman DNS on the `ov` bridge network. This requires the `ollama` container to be running on the same network:

```bash
ov start ollama       # Start Ollama server
ov start jupyter-colab-ml-finetuning   # Both on 'ov' network
```

The URL is configurable via the `OLLAMA_HOST` environment variable. Each notebook reads `os.getenv("OLLAMA_HOST", "http://ov-ollama:11434")`.

## Notebook Compatibility Notes

### ollama Python library env var gotcha

The `ollama` Python library creates its HTTP client at import time by reading `OLLAMA_HOST` from `os.environ`. Module-level functions (`ollama.list()`, `ollama.generate()`, etc.) are **bound methods** on this cached client. Simply setting a Python variable doesn't propagate to the library.

The notebooks use this pattern to handle both fresh kernels and re-runs:

```python
import os
OLLAMA_HOST = os.getenv("OLLAMA_HOST", "http://ov-ollama:11434")
os.environ["OLLAMA_HOST"] = OLLAMA_HOST

import ollama
import importlib; importlib.reload(ollama)  # Rebinds module functions
```

### Pydantic model changes (ollama >= 0.4)

The newer `ollama` library returns Pydantic models, not dicts. Model names are accessed via `.model` attribute (not `["name"]`), and the list is `.models` (not `.get("models", [])`):

```python
# Old (dict access)
model_names = [m.get("model", "") for m in models.get("models", [])]

# New (Pydantic attribute access)
model_names = [m.model for m in models.models]
```

### Default models

- 5 notebooks use `llama3.2:latest` (3.2B, Q4_K_M)
- `ollama_anthropic.ipynb` uses `ministral-3:3b-instruct-2512-q4_K_M`

Pull models before running: `ov shell ollama -c "ollama pull llama3.2"`

## Usage

```yaml
# images.yml
jupyter-colab-ml-finetuning:
  layers:
    - jupyter-colab-ml
    - notebook-templates
    - finetuning-notebooks
    - ollama-notebooks
    # ... other layers
```

```bash
# Deploy and start both services
ov config ollama && ov start ollama
ov config jupyter-colab-ml-finetuning && ov start jupyter-colab-ml-finetuning

# Pull a model, then open notebooks
ov shell ollama -c "ollama pull llama3.2"
# Open http://localhost:8888 -> navigate to ollama/
```

## Used In Images

- `/ov-images:jupyter-colab-ml-finetuning`

## Related Skills

- `/ov:layer` — data field documentation and layer authoring rules
- `/ov:config` — data provisioning during `ov config` setup
- `/ov:deploy` — volume backing configuration
- `/ov-layers:finetuning-notebooks` — sibling data layer pattern (Unsloth fine-tuning notebooks)
- `/ov-layers:notebook-templates` — sibling data layer pattern (starter notebooks)
- `/ov-layers:ollama` — the Ollama binary layer (server side)
- `/ov-images:ollama` — the standalone Ollama image (must be running for notebooks to connect)
- `/ov-images:jupyter-colab-ml-finetuning` — the image that includes this layer

## When to Use This Skill

Use when the user asks about:

- The ollama-notebooks layer or its contents
- Ollama API tutorial notebooks or integration examples
- How Jupyter notebooks connect to a separate Ollama container
- The `ollama` Python library env var or Pydantic API gotchas
- Which client libraries and API features are covered
