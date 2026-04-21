---
name: notebook-ollama
description: |
  Ollama integration notebook collection provisioned into the workspace volume at deploy time.
  6 Jupyter notebooks demonstrating Ollama via requests, OpenAI, ollama lib, Anthropic, HuggingFace, and GPU.
  Data-only layer — no packages, no services, no dependencies.
  Use when working with notebook-ollama, Ollama API tutorials, or Jupyter+Ollama integration.
---

# notebook-ollama -- Ollama integration notebook data layer

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none)* |
| Services | *(none)* |
| Volumes | `workspace` -> `~/workspace` (from jupyter) |
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
| `00_Ollama_Requests.ipynb` | `requests` | Raw REST API: list, show, generate, chat, stream, embed, copy, delete |
| `01_Ollama_GPU.ipynb` | `requests` | GPU verification: nvidia-smi, inference metrics, VRAM monitoring |
| `02_Ollama_OpenAI.ipynb` | `openai` | OpenAI-compatible API: completions, chat, multi-turn, stream, embed |
| `03_Ollama_Library.ipynb` | `ollama` | Native Python library: all API operations + model management |
| `04_Ollama_HuggingFace.ipynb` | `ollama` | HuggingFace GGUF model import, verification, inference |
| `05_Ollama_Anthropic.ipynb` | `anthropic` | Anthropic-compatible API: chat, system prompts, streaming, tool calling, vision |

**Manifest:** `notebooks.yaml` — structured catalog with title, description, and ordering.

## Network Connectivity

The notebooks default to `http://localhost:11434` for the Ollama API endpoint. Each notebook reads:

```python
OLLAMA_HOST = os.getenv("OLLAMA_HOST", "http://localhost:11434")
```

When the `ollama` image is deployed via `ov config ollama`, its `env_provides` automatically injects `OLLAMA_HOST=http://ov-ollama:11434` into the global `deploy.yml` env. Any container configured after ollama (or reconfigured with `--update-all`) automatically gets the correct Ollama endpoint — no manual environment setup needed.

**Setup:**
```bash
ov config ollama --update-all    # deploys ollama + propagates OLLAMA_HOST to all
ov start ollama
ov start jupyter-ml-notebook   # OLLAMA_HOST already set via env_provides
```

Both containers must be on the same `ov` Podman network for DNS resolution to work.

See `/ov:config` for `--update-all` and `/ov-layers:ollama` for env_provides details.

## Notebook Compatibility Notes

### ollama Python library env var gotcha

The `ollama` Python library creates its HTTP client at import time by reading `OLLAMA_HOST` from `os.environ`. Module-level functions (`ollama.list()`, `ollama.generate()`, etc.) are **bound methods** on this cached client. Simply setting a Python variable doesn't propagate to the library.

The notebooks use this pattern to handle both fresh kernels and re-runs:

```python
import os
OLLAMA_HOST = os.getenv("OLLAMA_HOST", "http://localhost:11434")
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
# image.yml
jupyter-ml-notebook:
  layers:
    - jupyter-ml
    - notebook-templates
    - notebook-finetuning
    - notebook-ollama
    # ... other layers
```

```bash
# Deploy and start both services
ov config ollama && ov start ollama
ov config jupyter-ml-notebook && ov start jupyter-ml-notebook

# Pull a model, then open notebooks
ov shell ollama -c "ollama pull llama3.2"
# Open http://localhost:8888 -> navigate to ollama/
```

## Used In Images

- `/ov-images:jupyter-ml-notebook`

## Related Skills

- `/ov:layer` — data field documentation and layer authoring rules
- `/ov:config` — data provisioning during `ov config` setup
- `/ov:deploy` — volume backing configuration
- `/ov-layers:notebook-finetuning` — sibling data layer pattern (Unsloth fine-tuning notebooks)
- `/ov-layers:notebook-templates` — sibling data layer pattern (starter notebooks)
- `/ov-layers:ollama` — the Ollama binary layer (server side)
- `/ov-images:ollama` — the standalone Ollama image (must be running for notebooks to connect)
- `/ov-images:jupyter-ml-notebook` — the image that includes this layer

## When to Use This Skill

Use when the user asks about:

- The notebook-ollama layer or its contents
- Ollama API tutorial notebooks or integration examples
- How Jupyter notebooks connect to a separate Ollama container
- The `ollama` Python library env var or Pydantic API gotchas
- Which client libraries and API features are covered

## Related

- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
