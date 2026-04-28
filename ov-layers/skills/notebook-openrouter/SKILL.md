---
name: notebook-openrouter
description: |
  OpenRouter API integration notebook collection provisioned into the workspace volume at deploy time.
  3 Jupyter notebooks demonstrating OpenRouter API basics, model discovery, and practical inference.
  Data-only layer with env_requires — first layer to use the env_requires feature.
  Use when working with notebook-openrouter, OpenRouter API tutorials, or Jupyter+OpenRouter integration.
---

# notebook-openrouter -- OpenRouter API integration data layer

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | *(none)* |
| Packages | *(none)* |
| Services | *(none)* |
| Volumes | `workspace` -> `~/workspace` (from jupyter) |
| Data | `data/openrouter` -> `workspace` volume, dest: `openrouter` |
| env_requires | `OPENROUTER_API_KEY` — API key for OpenRouter LLM inference |
| Install files | *(none)* |

## How It Works

This is a **data layer** with an **env_requires** declaration — it uses the `data:` field to map notebooks to a volume, and `env_requires:` to declare that the `OPENROUTER_API_KEY` environment variable must be present:

```yaml
info: "OpenRouter API integration notebook collection"

env_requires:
  - name: OPENROUTER_API_KEY
    description: "API key for OpenRouter LLM inference"

data:
  - src: data/openrouter
    volume: workspace
    dest: openrouter
```

At build time, the contents of `data/openrouter/` are staged into `/data/workspace/openrouter/` inside the image.

At deploy time, `ov config` copies the staged data into the workspace volume at `<workspace>/openrouter/`. The `dest: openrouter` field places the notebooks in a subdirectory rather than the volume root.

The `env_requires` declaration is stored as an OCI label (`org.overthinkos.env_requires`). At `ov config` time, if `OPENROUTER_API_KEY` is not present in the resolved environment, a warning is printed:
```
Warning: jupyter-ml-notebook requires OPENROUTER_API_KEY (API key for OpenRouter LLM inference) — not set
```

This is the first layer in the project to use the `env_requires` feature.

## Included Notebooks

3 Jupyter notebooks covering OpenRouter API usage with the free model `qwen/qwen3.6-plus:free`:

| Notebook | Topic | Features |
|----------|-------|----------|
| `00_OpenRouter_Basics.ipynb` | API Basics | Authentication, simple chat completion, multi-turn conversation, error handling, usage/cost tracking |
| `01_OpenRouter_Models.ipynb` | Model Selection | List all models, filter free models, inspect model details (pricing, context, modality), compare responses across models |
| `02_OpenRouter_Inference.ipynb` | Practical Inference | Structured JSON output, text summarization, code generation (with exec), reasoning with thinking tokens, translation pipeline |

**Manifest:** `notebooks.yaml` — structured catalog with title, description, and ordering.

## Free Tier Rate Limit Gotcha

The free model `qwen/qwen3.6-plus:free` has strict rate limits (~1 request per minute). All notebooks include an `openrouter_chat()` / `chat()` retry helper that handles 429 responses with exponential backoff:

```python
def openrouter_chat(model, messages, retries=5, delay=15):
    for attempt in range(retries):
        response = requests.post(...)
        if response.status_code == 429:
            wait = delay * (attempt + 1)
            print(f"Rate limited, waiting {wait}s...")
            time.sleep(wait)
            continue
        response.raise_for_status()
        return response.json()
```

The retry helper also handles transient 502 upstream errors from Alibaba (the Qwen provider) where the response is 200 but missing the `choices` key.

## Providing the API Key

The `OPENROUTER_API_KEY` must be injected into the container environment:

```bash
# Via ov config (persisted in quadlet)
ov config jupyter-ml-notebook -e OPENROUTER_API_KEY=sk-or-v1-...

# Via workspace .env file
echo "OPENROUTER_API_KEY=sk-or-v1-..." >> ~/project/.env

# Via direnv .secrets (GPG-encrypted)
ov secrets gpg set OPENROUTER_API_KEY sk-or-v1-...
```

## Usage

```yaml
# image.yml
jupyter-ml-notebook:
  layers:
    - jupyter-ml
    - notebook-templates
    - notebook-finetuning
    - notebook-ollama
    - notebook-llm-on-supercomputers
    - notebook-openrouter
    # ... other layers
```

```bash
# Deploy with API key
ov config jupyter-ml-notebook -e OPENROUTER_API_KEY=sk-or-v1-...
ov start jupyter-ml-notebook
# Open http://localhost:8888 -> navigate to openrouter/
```

## Used In Images

- `/ov-images:jupyter-ml-notebook`

## Related Skills

- `/ov:layer` — data field documentation, env_requires/env_accepts field documentation
- `/ov:config` — data provisioning and env_requires warning during `ov config` setup
- `/ov:deploy` — volume backing configuration
- `/ov-layers:notebook-ollama` — sibling data layer pattern (Ollama integration notebooks)
- `/ov-layers:notebook-finetuning` — sibling data layer pattern (Unsloth fine-tuning notebooks)
- `/ov-layers:notebook-templates` — sibling data layer pattern (starter notebooks)
- `/ov-layers:notebook-llm-on-supercomputers` — sibling data layer pattern (TU Wien course notebooks)
- `/ov-images:jupyter-ml-notebook` — the image that includes this layer

## When to Use This Skill

Use when the user asks about:

- The notebook-openrouter layer or its contents
- OpenRouter API tutorial notebooks or integration examples
- How to provide `OPENROUTER_API_KEY` to Jupyter containers
- The `env_requires` feature (this is the first real consumer)
- Free tier rate limits on `qwen/qwen3.6-plus:free`
- Structured output, reasoning with thinking tokens, or model comparison via OpenRouter

## Related

- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
