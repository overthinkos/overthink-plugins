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
| Install files | `pixi.toml`, `build.sh`, `user.yml` |
| RPM packages | `alsa-lib`, `portaudio` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `HERMES_HOME` | `/opt/data` |
| `TERMINAL_ENV` | `local` |

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

## When to Use This Skill

**MUST be invoked** when the task involves the hermes layer, Hermes Agent setup, hermes service configuration, hermes Python dependencies, or the hermes entrypoint. Invoke this skill BEFORE reading source code or launching Explore agents.
