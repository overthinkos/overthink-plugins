---
name: hermes
description: |
  Hermes self-improving AI agent by Nous Research with voice, messaging, and tool-calling.
  MUST be invoked before any work involving: the hermes layer, Hermes Agent configuration,
  hermes service setup, or hermes Python/npm dependencies.
---

# hermes -- Self-improving AI agent

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs`, `supervisord`, `ripgrep`, `ffmpeg`, `pipewire` |
| Volumes | `data` -> `/opt/data` |
| Aliases | `hermes` -> `hermes`, `hermes-agent` -> `hermes-agent` |
| Services | `hermes` (supervisord, autostart), `hermes-whatsapp` (supervisord, manual) |
| MCP accepts | `jupyter-colab` |
| Install files | `pixi.toml`, `build.sh`, `user.yml` |
| RPM packages | `alsa-lib`, `portaudio` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `HERMES_HOME` | `/opt/data` |
| `TERMINAL_ENV` | `local` |

## Optional Environment Variables (env_accepts)

These env vars are declared via `env_accepts:` in `layer.yml` — hermes can use any LLM provider, so none are required:

| Variable | Description |
|----------|-------------|
| `OPENROUTER_API_KEY` | API key for OpenRouter LLM inference |
| `OLLAMA_API_KEY` | API key for Ollama Cloud inference (https://ollama.com) |
| `OLLAMA_HOST` | Local Ollama server URL (auto-injected by ollama layer `env_provides`) |
| `HERMES_MODEL` | Override default Hermes model (default depends on detected provider) |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token for messaging |
| `SLACK_BOT_TOKEN` | Slack bot token |
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `OV_MCP_SERVERS` | JSON array of MCP servers (auto-injected by mcp_provides layers) |

Provide via `ov config hermes -e OLLAMA_API_KEY=...` or workspace `.secrets` / `.env` file.

## Automatic LLM Provider Configuration

The `hermes-entrypoint` performs a **single-phase, first-start-only** configuration that registers ALL available LLM providers AND MCP servers in one pass. Guarded by a `# ov:auto-configured` sentinel in `config.yaml`. To reconfigure: delete `config.yaml` and restart. API keys are synced to `.env` on every start to handle rotation.

| Priority | Env var | Provider | Default model |
|----------|---------|----------|---------------|
| 1 | `OLLAMA_HOST` | Local Ollama (`custom`) | `qwen2.5-coder:32b` |
| 2 | `OLLAMA_API_KEY` | Ollama Cloud (`custom`) | `kimi-k2.5:cloud` |
| 3 | `OPENROUTER_API_KEY` | OpenRouter (built-in) | `qwen/qwen3.6-plus:free` |

**How it works:**
- **First start:** Registers ALL providers whose env vars are set into `config.yaml` as `custom_providers` entries, and configures any discovered MCP servers from `OV_MCP_SERVERS`. Priority determines only the default model and auxiliary task routing. Writes a `# ov:auto-configured` sentinel to prevent re-patching after user customization
- **Every start:** Syncs API keys to `.env` to handle key rotation
- Override the default model with `HERMES_MODEL` env var
- Switch between registered providers mid-session: `/model custom:ollama-cloud:kimi-k2.5:cloud` or `hermes chat --provider openrouter`
- To force full reconfigure: delete `/opt/data/config.yaml`, update env vars, restart

## MCP Server Discovery

The hermes entrypoint auto-discovers MCP servers from the `OV_MCP_SERVERS` environment variable at first start. Servers are configured in `config.yaml` under the `mcp_servers:` key (hermes native YAML map format).

**Diagnostics:**
- `hermes mcp list` -- shows registered MCP servers
- `hermes mcp test <name>` -- tests connection to a specific server

**Runtime:**
- Tools are registered as `mcp_<server_name>_<tool_name>`
- Reload MCP servers at runtime: `/reload-mcp` in interactive chat

**Example:** The `jupyter-colab` layer provides 13 tools (list_notebooks, execute_cell, etc.) via its MCP server at `http://<container>:8888/mcp`.

## Architecture

This is a **Tier 2 environment-owner layer** with `pixi.toml` defining the Python 3.13 environment. Follows the **selkies build.sh pattern**: the `build.sh` script runs in the pixi builder stage (which has gcc, nodejs, npm) to clone the hermes-agent repo, pip install it, and set up the WhatsApp bridge.

### Build Pipeline

1. **pixi.toml** -- Python 3.13 + all hermes-agent `[all]` PyPI dependencies (openai, anthropic, httpx, rich, pydantic, telegram, discord, slack, faster-whisper, sounddevice, elevenlabs, mcp, and more)
2. **build.sh** -- Runs in builder stage:
   - `git clone --depth 1` hermes-agent from GitHub
   - `pip install --no-deps .` into pixi env (non-editable)
   - `npm install` in project root (agent-browser, camoufox-browser)
   - `npm install` in `scripts/whatsapp-bridge/`
   - Copies full source tree to `$HOME/hermes-agent` for runtime files
3. **user.yml** -- Copies `hermes-entrypoint` wrapper to `~/.local/bin/`

### Runtime Entrypoint

The `hermes-entrypoint` script (run by supervisord) initializes the data volume on first start:
- Creates `$HERMES_HOME/{cron,sessions,logs,hooks,memories,skills}`
- Copies `.env.example`, `config.yaml`, `SOUL.md` defaults if not present
- Auto-configures LLM provider from environment (see Automatic LLM Provider Configuration above)
- Syncs bundled skills via `skills_sync.py`
- Execs `hermes`

### npm Dependencies

Agent-browser and camoufox-browser are installed in the **project-level** `node_modules/` (via `build.sh`), not globally. Hermes expects these as local imports from its source tree at `~/hermes-agent/`.

### WhatsApp Bridge

The WhatsApp bridge is a separate Node.js service with `autostart=false` in supervisord. Enable via:
```bash
ov service start hermes hermes-whatsapp
```

## Excluded Dependencies

- `matrix` extra -- upstream-broken libolm (archived, C++ errors with Clang 21+)
- `rl` extra -- requires git dependencies (atropos, tinker)
- `yc-bench` extra -- requires git dependency

## Usage

```yaml
# images.yml
hermes:
  base: fedora
  layers:
    - agent-forwarding
    - hermes
    - dbus
    - ov
```

## Used In Images

- `/ov-images:hermes` -- headless agent (no browser)
- `/ov-images:hermes-playwright` -- with Playwright Chromium

## Related Layers

- `/ov-layers:nodejs` -- Node.js runtime dependency
- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:ripgrep` -- fast search dependency
- `/ov-layers:ffmpeg` -- audio/video processing (negativo17 nonfree codecs)
- `/ov-layers:pipewire` -- audio support for voice features
- `/ov-layers:hermes-playwright` -- optional Playwright Chromium browser
- `/ov-layers:jupyter-colab` -- MCP server provider (mcp_provides)

## When to Use This Skill

**MUST be invoked** when the task involves the hermes layer, Hermes Agent setup, hermes service configuration, hermes Python dependencies, or the hermes entrypoint. Invoke this skill BEFORE reading source code or launching Explore agents.
