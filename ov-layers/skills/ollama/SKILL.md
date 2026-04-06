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

## Service Environment (injected into other containers)

| Variable | Template Value | Resolved Example |
|----------|---------------|-----------------|
| `OLLAMA_HOST` | `http://{{.ContainerName}}:11434` | `http://ov-ollama:11434` |

When `ov config ollama` runs, `OLLAMA_HOST` is automatically injected into the global `deploy.yml` env. All other deployed containers receive this variable, enabling automatic Ollama service discovery. Use `ov config ollama --update-all` to propagate to already-deployed services immediately.

See `/ov:layer` for `service_env` field docs and `/ov:config` for `--update-all`.

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
