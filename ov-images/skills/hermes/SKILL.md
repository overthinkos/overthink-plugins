---
name: hermes
description: |
  Headless Hermes AI agent image. Self-improving agent with voice, messaging,
  and tool-calling — no browser. Use when working with the headless hermes deployment.
  MUST be invoked before building, deploying, configuring, or troubleshooting the hermes image.
---

# hermes

Headless Hermes AI agent by Nous Research — voice, messaging, tool-calling, skill learning. No browser.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, hermes, dbus, ov |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `pixi` -> `python` -> `supervisord` (transitive)
3. `nodejs` (transitive via hermes)
4. `ripgrep` (transitive via hermes)
5. `ffmpeg` (transitive via hermes, negativo17 nonfree codecs)
6. `pipewire` (transitive via hermes, audio for voice features)
7. `hermes` -- agent on supervisord, data volume at `/opt/data`
8. `agent-forwarding` (gnupg + direnv + ssh-client)
9. `dbus` -- D-Bus session bus
10. `ov` -- ov CLI

## Services

| Service | Status | Description |
|---------|--------|-------------|
| `hermes` | autostart | Main Hermes agent process |
| `hermes-whatsapp` | manual | WhatsApp bridge (Node.js) |
| `dbus` | autostart | D-Bus session bus |
| `pipewire` | autostart | Audio server for voice |

## Quick Start

```bash
ov build hermes
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

## MCP Server Discovery

When co-deployed with services that declare `mcp_provides` (e.g., jupyter-colab), hermes auto-discovers and connects to their MCP servers at first start. The `OV_MCP_SERVERS` JSON env var is injected by `ov config` and the entrypoint writes the servers into `config.yaml` under `mcp_servers:`.

```bash
# Verify MCP connection
ov shell hermes -c "hermes mcp list"                    # Shows registered servers
ov shell hermes -c "hermes mcp test jupyter-colab"      # Tests connection (expects 13 tools)

# Use MCP tools in chat
ov shell hermes -c "hermes chat -q 'Use list_notebooks to list notebooks'"
```

Tools are registered as `mcp_<server_name>_<tool_name>`. Reload at runtime: `/reload-mcp`.

## Key Layers

- `/ov-layers:hermes` -- core agent layer (pixi.toml, build.sh, service, volumes)
- `/ov-layers:ffmpeg` -- nonfree audio/video codecs
- `/ov-layers:pipewire` -- audio for voice features (sounddevice, faster-whisper, edge-tts)

## Related Images

- `/ov-images:hermes-playwright` -- adds Playwright Chromium for browser automation
- Deploy `hermes-full` alongside `selkies-desktop` and `jupyter-colab` as separate pods for desktop + agent + notebook workflows
- `/ov-images:openclaw` -- alternative AI gateway (OpenClaw vs Hermes)

## Verification

After `ov start`:
```bash
ov status hermes                    # container running
ov service status hermes            # all services RUNNING
ov shell hermes -c "hermes --version"    # Hermes Agent v0.7.0
ov shell hermes -c "python -c 'import openai; import anthropic; print(\"OK\")'"
ov shell hermes -c "ffmpeg -version"     # nonfree codecs present
ov shell hermes -c "cd ~/hermes-agent && node -e \"require('agent-browser')\""
```

## When to Use This Skill

**MUST be invoked** when the task involves the headless hermes image, deploying Hermes Agent, or comparing hermes variants. Invoke this skill BEFORE reading source code or launching Explore agents.
