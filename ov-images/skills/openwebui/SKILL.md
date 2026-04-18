---
name: openwebui
description: |
  Open WebUI image with auto-configured LLM providers, MCP servers, and Jupyter on port 8080.
  MUST be invoked before building, deploying, configuring, or troubleshooting the openwebui image.
---

# Image: openwebui

Open WebUI with auto-configured LLM providers (Ollama, OpenRouter), MCP server discovery, and Jupyter code execution. No manual setup needed — secrets auto-managed via `ov secrets`.

## Definition

```yaml
openwebui:
  base: fedora
  layers:
    - agent-forwarding
    - openwebui
    - dbus
    - ov
  ports:
    - "8080:8080"
```

## Layer Stack

| Layer | Purpose |
|-------|---------|
| `agent-forwarding` | SSH + GPG agent forwarding into container |
| `openwebui` | Open WebUI with auto-config entrypoint |
| `dbus` | D-Bus session bus |
| `ov` | Overthink CLI for in-container management |

## Quick Start

```bash
ov image build openwebui
ov config openwebui -e OPENROUTER_API_KEY=sk-or-xxx
ov start openwebui
# Open http://localhost:8080
```

## Deployment Patterns

### Standalone with GPG secrets (recommended)
```bash
ov secrets gpg setup
ov secrets gpg set OPENROUTER_API_KEY sk-or-xxx
ov image build openwebui
ov config openwebui --env-file .secrets
ov start openwebui
```

### With local Ollama
```bash
ov config ollama
ov config openwebui --env-file .secrets --update-all
ov start ollama openwebui
```

### Full workstation (Ollama + Jupyter + Browser)
```bash
ov config ollama
ov config jupyter --update-all
ov config selkies-desktop --update-all
ov config openwebui --env-file .secrets --update-all
ov start ollama jupyter selkies-desktop openwebui
```

## Secrets Management

```bash
# Tier 1: Auto-generated infrastructure secrets
ov secrets list ov/openwebui
ov secrets get ov/openwebui webui-secret-key
ov secrets set ov/openwebui admin-password --generate

# Tier 2: User API keys (GPG-encrypted)
ov secrets gpg set OPENROUTER_API_KEY sk-or-new-key
ov secrets gpg show
ov config openwebui --env-file .secrets --update-all
```

## Cross-Container Service Discovery

Deploy alongside provider containers for full functionality:

```bash
# 1. Deploy ollama (provides OLLAMA_HOST)
ov config ollama
ov start ollama

# 2. Deploy jupyter (provides jupyter MCP server)
ov config jupyter --update-all
ov start jupyter

# 3. Deploy openwebui (consumes both)
ov config openwebui --env-file .secrets --update-all
ov start openwebui
```

Open WebUI receives:
- `OLLAMA_BASE_URL=http://ov-ollama:11434` — local LLM inference
- `TOOL_SERVER_CONNECTIONS=[...]` — MCP servers (jupyter + chrome-devtools)
- `CODE_EXECUTION_ENGINE=jupyter` — code execution via Jupyter

## Key Layers

- `/ov-layers:openwebui` — Auto-config entrypoint, secrets, env_accepts, TOOL_SERVER_CONNECTIONS format
- `/ov-layers:agent-forwarding` — SSH/GPG forwarding

## Related Images

- `/ov-images:jupyter` — deploy alongside for MCP notebooks and code execution
- `/ov-images:ollama` — deploy alongside for local LLM inference
- `/ov-images:hermes` — alternative AI frontend (CLI agent vs web UI)
- `/ov-images:selkies-desktop` — deploy alongside for shared Chrome browser

## Verification

```bash
ov status openwebui
ov service status openwebui                    # openwebui: RUNNING
curl -s -o /dev/null -w '%{http_code}' http://localhost:8080   # 200
ov shell openwebui -c "open-webui version"

# Verify secrets
podman secret ls | grep openwebui              # webui-secret-key, admin-password

# Verify MCP (inside container process)
podman exec ov-openwebui cat /proc/3/environ | tr '\0' '\n' | grep TOOL_SERVER_CONNECTIONS
```

## Test Coverage

Latest `ov test openwebui` run: **24 passed, 0 failed, 0 skipped**.
Covers: openwebui entrypoint script presence, pixi python + ov binary,
and deploy-scope: service up, port reachable on `127.0.0.1:${HOST_PORT:8080}`,
HTTP 200 on `/` (30-second timeout for first-request startup), admin
email env var injected. See `/ov:test` for the framework.

## Related Skills

- `/ov-layers:openwebui` — layer authoring
- `/ov:test` — declarative testing framework
- `/ov:secrets` — WEBUI_ADMIN_PASSWORD + provider API keys
- `/ov:config` — `-e WEBUI_ADMIN_EMAIL=...` deploy-time env setup

## When to Use This Skill

**MUST be invoked** before building, deploying, configuring, or troubleshooting the openwebui image.
