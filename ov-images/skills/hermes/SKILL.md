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
ov config hermes
ov start hermes
ov shell hermes -c "hermes chat"
```

## Configuration

API keys and settings are provided via environment variables at deploy time:
```bash
# Via ov config env vars
ov config hermes -e OPENROUTER_API_KEY=sk-xxx -e TELEGRAM_BOT_TOKEN=xxx

# Or via .env file in the data volume
ov shell hermes -c "vi /opt/data/.env"
```

The data volume (`/opt/data`) persists: sessions, skills, memories, logs, config, and `.env`.

## Key Layers

- `/ov-layers:hermes` -- core agent layer (pixi.toml, build.sh, service, volumes)
- `/ov-layers:ffmpeg` -- nonfree audio/video codecs
- `/ov-layers:pipewire` -- audio for voice features (sounddevice, faster-whisper, edge-tts)

## Related Images

- `/ov-images:hermes-playwright` -- adds Playwright Chromium for browser automation
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
