---
name: hermes-full
description: |
  Full-featured standalone Hermes agent with AI CLIs, dev tools, DevOps tools, and ov.
  MUST be invoked before building, deploying, configuring, or troubleshooting the hermes-full image.
---

# Image: hermes-full

Full-featured standalone Hermes AI agent. No browser or desktop — designed for cross-container deployment alongside `selkies-desktop` (shared Chrome via `BROWSER_CDP_URL`) and `jupyter-colab` (MCP notebooks via `OV_MCP_SERVERS`).

## Definition

```yaml
hermes-full:
  base: fedora
  layers:
    - agent-forwarding
    - hermes-full      # hermes + claude-code + codex + gemini + dev-tools + devops-tools + ov
    - dbus
```

## Layer Stack

| Layer | Purpose |
|-------|---------|
| `agent-forwarding` | SSH + GPG agent forwarding into container |
| `hermes` | AI agent with LLM auto-config, MCP, browser tools |
| `claude-code` | Anthropic Claude Code CLI |
| `codex` | OpenAI Codex CLI |
| `gemini` | Google Gemini CLI |
| `dev-tools` | bat, ripgrep, neovim, gh, direnv, fd-find, htop, podman-compose, etc. |
| `devops-tools` | AWS CLI, Scaleway, kubectx/kubens, OpenTofu, wrangler, bind-utils, jq, rsync |
| `ov` | Overthink CLI for in-container management |
| `dbus` | D-Bus session bus |

## Cross-Container Service Discovery

Deploy alongside provider containers for full functionality:

```bash
# 1. Deploy selkies-desktop (provides BROWSER_CDP_URL)
ov config selkies-desktop
ov start selkies-desktop

# 2. Deploy jupyter-colab (provides jupyter-colab MCP server)
ov config jupyter-colab --update-all
ov start jupyter-colab

# 3. Deploy hermes-full (consumes both)
ov config hermes-full -e OLLAMA_API_KEY=... --update-all
ov start hermes-full
```

Hermes receives:
- `BROWSER_CDP_URL=http://ov-selkies-desktop:9222` — controls desktop Chrome
- `OV_MCP_SERVERS=[{"name":"jupyter-colab","url":"http://ov-jupyter-colab:8888/mcp"}]` — notebook manipulation

## Verification

```bash
ov status hermes-full
ov service status hermes-full                    # hermes: RUNNING
ov shell hermes-full -c "hermes --version"
ov shell hermes-full -c "claude --version"
ov shell hermes-full -c "codex --version"
ov shell hermes-full -c "gemini --version"
ov shell hermes-full -c "ov version"
ov shell hermes-full -c "echo BROWSER_CDP_URL=\$BROWSER_CDP_URL"
ov shell hermes-full -c "echo OV_MCP_SERVERS=\$OV_MCP_SERVERS"
```

## Related Layers

- `/ov-layers:hermes-full` — Metalayer composition details
- `/ov-layers:hermes` — Core agent (env_accepts, browser dispatch, LLM config)
- `/ov-layers:chrome` — Provides `BROWSER_CDP_URL` (from selkies-desktop)

## Related Images

- `/ov-images:hermes` — Minimal standalone hermes (no AI CLIs, no dev tools)
- `/ov-images:hermes-playwright` — Hermes with local Playwright Chromium
- `/ov-images:selkies-desktop` — Desktop with Chrome (cross-container browser provider)
- `/ov-images:jupyter-colab` — JupyterLab with MCP (cross-container MCP provider)

## When to Use This Skill

**MUST be invoked** before building, deploying, configuring, or troubleshooting the hermes-full image.
