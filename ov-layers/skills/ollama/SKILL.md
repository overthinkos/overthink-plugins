---
name: ollama
description: |
  Ollama LLM server on port 11434 with CUDA GPU support and model persistence.
  Use when working with Ollama, LLM serving, or local AI model inference.
---

# ollama -- Local LLM inference server

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `cuda`, `supervisord` |
| Ports | 11434 |
| Volumes | `models` -> `~/.ollama` |
| Aliases | `ollama` -> `ollama` |
| Service | `ollama` (supervisord) |
| Install files | `root.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `OLLAMA_HOST` | `0.0.0.0` |
| `OLLAMA_MODELS` | `~/.ollama/models` |

## Usage

```yaml
# images.yml
ollama:
  layers:
    - ollama
```

```bash
ov alias install ollama    # install host 'ollama' command
ollama run llama3          # uses the alias
```

## Used In Images

- `/ov-images:ollama`
- `/ov-images:openclaw-ollama`
- `/ov-images:openclaw-ollama-sway-browser`

## Related Layers

- `/ov-layers:cuda` -- CUDA toolkit dependency
- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:openclaw` -- AI gateway that can use Ollama as backend

## When to Use This Skill

Use when the user asks about:

- Ollama setup or model management
- LLM serving or inference
- Port 11434 configuration
- Ollama model storage volume
- The `ollama` host alias
