---
name: hermes
description: |
  Full-featured standalone Hermes AI agent with AI CLIs, dev tools, DevOps tools, and ov.
  MUST be invoked before building, deploying, configuring, or troubleshooting the hermes image.
---

# Image: hermes

Full-featured standalone Hermes AI agent. No browser or desktop — designed for cross-container deployment alongside `selkies-desktop` (shared Chrome via `BROWSER_CDP_URL`) and `jupyter` (MCP notebooks via `OV_MCP_SERVERS`).

## Definition

```yaml
hermes:
  base: fedora
  layers:
    - agent-forwarding
    - hermes-full      # hermes + claude-code + codex + gemini + dev-tools + devops-tools + ov + tmux
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
| `tmux` | Terminal multiplexer for persistent sessions |
| `dbus` | D-Bus session bus |

## Quick Start

```bash
ov image build hermes
ov config hermes -e OLLAMA_API_KEY=your-key   # or OPENROUTER_API_KEY
ov start hermes
ov shell hermes -c "hermes chat"
```

## Configuration

The hermes entrypoint performs a **single-phase, first-start-only** auto-configuration of LLM providers and MCP servers. Guarded by `# ov:auto-configured` sentinel in `config.yaml`. To reconfigure: delete `/opt/data/config.yaml` and restart. API keys are synced to `.env` on every start (handles rotation).

LLM provider based on which env var is set:

| Priority | Env var | Provider | Default model |
|----------|---------|----------|---------------|
| 1 | `OLLAMA_HOST` | Local Ollama | `qwen2.5-coder:32b` |
| 2 | `OLLAMA_API_KEY` | Ollama Cloud | `kimi-k2.5:cloud` |
| 3 | `OPENROUTER_API_KEY` | OpenRouter | `qwen/qwen3.6-plus:free` |

Override the default model with `HERMES_MODEL` env var. Additional `env_accepts`:

| Variable | Description |
|----------|-------------|
| `TELEGRAM_BOT_TOKEN` | Telegram bot token for messaging |
| `SLACK_BOT_TOKEN` | Slack bot token |
| `DISCORD_BOT_TOKEN` | Discord bot token |

```bash
# Ollama Cloud
ov config hermes -e OLLAMA_API_KEY=your-key

# OpenRouter with custom model
ov config hermes -e OPENROUTER_API_KEY=sk-or-xxx -e HERMES_MODEL=google/gemini-2.5-flash

# Local Ollama (OLLAMA_HOST auto-injected by ov config ollama --update-all)
ov config hermes

# Or via .env file in the data volume
ov shell hermes -c "vi /opt/data/.env"
```

The data volume (`/opt/data`) persists: sessions, skills, memories, logs, config, and `.env`.

## Cross-Container Service Discovery

Deploy alongside provider containers for full functionality:

```bash
# 1. Deploy selkies-desktop (provides BROWSER_CDP_URL)
ov config selkies-desktop
ov start selkies-desktop

# 2. Deploy jupyter (provides jupyter MCP server)
ov config jupyter --update-all
ov start jupyter

# 3. Deploy hermes (consumes both)
ov config hermes -e OLLAMA_API_KEY=... --update-all
ov start hermes
```

Hermes receives:
- `BROWSER_CDP_URL=http://ov-selkies-desktop:9222` — controls desktop Chrome
- `OV_MCP_SERVERS=[{"name":"jupyter","url":"http://ov-jupyter:8888/mcp"},{"name":"chrome-devtools","url":"http://ov-selkies-desktop:9224/mcp"}]` — notebook manipulation + browser DevTools MCP

## MCP Server Discovery

When co-deployed with services that declare `mcp_provides` (e.g., jupyter), hermes auto-discovers and connects to their MCP servers at first start. The `OV_MCP_SERVERS` JSON env var is injected by `ov config` and the entrypoint writes the servers into `config.yaml` under `mcp_servers:`.

```bash
# Verify MCP connection
ov shell hermes -c "hermes mcp list"                    # Shows registered servers
ov shell hermes -c "hermes mcp test jupyter"      # Tests connection (expects 13 tools)
```

## Key Layers

- `/ov-layers:hermes-full` — Metalayer composition details
- `/ov-layers:hermes` — Core agent (env_accepts, browser dispatch, LLM config)
- `/ov-layers:chrome` — Provides `BROWSER_CDP_URL` (from selkies-desktop)
- `/ov-layers:chrome-devtools-mcp` — Chrome DevTools MCP server on port 9224 (from selkies-desktop)

## Related Images

- `/ov-images:hermes-playwright` — Hermes with local Playwright Chromium
- `/ov-images:selkies-desktop` — Desktop with Chrome (cross-container browser provider)
- `/ov-images:jupyter` — JupyterLab with MCP (cross-container MCP provider)

## Verification

```bash
ov status hermes
ov service status hermes                    # hermes: RUNNING
ov shell hermes -c "hermes --version"
ov shell hermes -c "claude --version"
ov shell hermes -c "codex --version"
ov shell hermes -c "gemini --version"
ov shell hermes -c "ov version"
ov shell hermes -c "echo BROWSER_CDP_URL=\$BROWSER_CDP_URL"
ov shell hermes -c "echo OV_MCP_SERVERS=\$OV_MCP_SERVERS"
ov shell hermes -c "hermes mcp list"              # Should show chrome-devtools, jupyter
```

## Test Coverage

Latest `ov test hermes` run: **50 passed, 0 failed, 0 skipped**.
Covers all 4 AI CLIs at `${HOME}/.npm-global/bin/{claude,codex,gemini}`
+ pixi's `hermes` at `${HOME}/.pixi/envs/default/bin/hermes`, plus
dev-tools (rg, bat, gh, fastfetch, nvim, htop) and devops-tools (aws,
scw, kubectx, kubens, tofu, jq). Deploy-scope: pipewire + hermes
services up, `/opt/data` volume mounted. Uses `supervisorctl pid` for
liveness (hermes-whatsapp is autostart=false — see `/ov:test` Gotcha #4).

## Related Skills

- `/ov-layers:hermes-full`, `/ov-layers:hermes`, `/ov-layers:claude-code`,
  `/ov-layers:codex`, `/ov-layers:gemini`, `/ov-layers:dev-tools`,
  `/ov-layers:devops-tools`
- `/ov:test` — declarative testing framework + supervisord gotchas
- `/ov:config` — `OV_MCP_SERVERS` auto-discovery + secret provisioning
- `/ov-images:selkies-desktop` — companion for shared browser (CDP)
- `/ov-images:jupyter` — MCP notebook tools auto-discovered

## When to Use This Skill

**MUST be invoked** before building, deploying, configuring, or troubleshooting the hermes image.
