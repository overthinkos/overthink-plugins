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
| Install files | `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `OLLAMA_HOST` | `0.0.0.0` |
| `OLLAMA_MODELS` | `~/.ollama/models` |

## Service Environment (injected into other containers)

| Variable | Template Value | Resolved Example |
|----------|---------------|-----------------|
| `OLLAMA_HOST` | `http://{{.ContainerName}}:11434` | `http://ov-ollama:11434` |

Pod-aware: same-container consumers receive `http://localhost:11434`, cross-container consumers receive `http://ov-ollama:11434`. When `ov config ollama` runs, `OLLAMA_HOST` is automatically injected into the global `deploy.yml` env. Use `ov config ollama --update-all` to propagate to already-deployed services immediately.

See `/ov:layer` for `env_provides` field docs and `/ov:config` for `--update-all`.

## Usage

```yaml
# image.yml
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

## Cross-Container Integration

The `env_provides` mechanism makes `OLLAMA_HOST` available to all containers. The `hermes` layer auto-detects this variable and configures itself to use local Ollama as its LLM provider (highest priority in the auto-detection chain: `OLLAMA_HOST` > `OLLAMA_API_KEY` > `OPENROUTER_API_KEY`). See `/ov-layers:hermes` for details on the auto-provider-configuration.

## Tests

The layer ships 3 declarative checks embedded in the `org.overthinkos.eval`
OCI label (see `/ov:test` for the full schema):

- **Build-scope** (run under `ov eval image`):
  - `ollama-binary` — `/usr/bin/ollama` exists
- **Deploy-scope** (run under `ov test` against a live service; uses
  `${HOST_PORT:11434}` / `${CONTAINER_IP}` so deploy-time port remapping
  works unchanged):
  - `ollama-tags-api` — `GET http://${CONTAINER_IP}:${HOST_PORT:11434}/api/tags` returns 200
  - `ollama-version` — `ollama --version` stdout matches `^ollama version`

## Related Layers

- `/ov-layers:cuda` -- CUDA toolkit dependency
- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:openclaw` -- AI gateway that can use Ollama as backend
- `/ov-layers:hermes` -- AI agent that auto-detects `OLLAMA_HOST` for local Ollama provider

## Related Commands

- `/ov:config` — Deploy with quadlet (secrets, volumes, env_provides injection)
- `/ov:start` — Start the Ollama service
- `/ov:service` — Manage Ollama service inside container

## When to Use This Skill

Use when the user asks about:

- Ollama setup or model management
- LLM serving or inference
- Port 11434 configuration
- Ollama model storage volume
- The `ollama` host alias
