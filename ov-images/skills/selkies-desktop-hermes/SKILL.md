---
name: selkies-desktop-hermes
description: |
  Combined Selkies streaming desktop + Hermes AI agent image.
  Use when working with the selkies-desktop-hermes image.
---

# selkies-desktop-hermes

Browser-accessible Wayland desktop streamed via Selkies/pixelflux WebSocket at `https://localhost:3000`, with the Hermes AI agent running alongside the desktop.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora-nonfree |
| Layers | agent-forwarding, selkies-desktop, hermes, claude-code, codex, gemini, dbus, ov |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Definition

```yaml
selkies-desktop-hermes:
  base: fedora-nonfree
  layers:
    - agent-forwarding
    - selkies-desktop
    - hermes
    - claude-code
    - codex
    - gemini
    - dbus
    - ov
  ports:
    - "3000:3000"
    - "9222:9222"
  platforms:
    - linux/amd64
```

## Base

`fedora-nonfree` — Fedora 43 with RPM Fusion (required for Selkies multimedia codecs). Hermes normally uses plain `fedora`, but `fedora-nonfree` is a superset. FFmpeg is provided via the `ffmpeg` layer (negativo17 nonfree build) pulled in by the `hermes` layer dependency.

## Full Layer Stack

1. `fedora-nonfree` (Fedora 43 + RPM Fusion)
2. `pixi` → `python` → `supervisord` (transitive via hermes)
3. `nodejs` (transitive via hermes)
4. `ripgrep` (transitive via hermes)
5. `ffmpeg` (transitive via hermes — negativo17 nonfree codecs)
6. `pipewire` (transitive via selkies-desktop and hermes — shared audio server)
7. `selkies-desktop` metalayer (pipewire + chrome + labwc + waybar-labwc + desktop-fonts + swaync + pavucontrol + wl-tools + wl-screenshot-pixelflux + wl-overlay + wl-record-pixelflux + a11y-tools + xterm + tmux + asciinema + fastfetch + selkies)
8. `hermes` — AI agent on supervisord, data volume at `/opt/data`
9. `claude-code` — Claude Code CLI (npm, via nodejs already present)
10. `codex` — OpenAI Codex CLI (npm)
11. `gemini` — Google Gemini CLI (npm)
12. `agent-forwarding` (gnupg + direnv + ssh-client)
13. `dbus` — D-Bus session bus
14. `ov` — ov CLI

## Services

| Service | Priority | Description |
|---------|----------|-------------|
| `dbus` | 2 | D-Bus session bus |
| `pipewire` | 5 | Audio server (shared by Selkies streaming + Hermes voice) |
| `selkies` | 8 | Wayland capture + H.264/Opus streaming via pixelflux |
| `labwc` | 12 | Wayland compositor (nested in pixelflux) |
| `swaync` | 14 | Notification daemon |
| `waybar` | 15 | Status bar |
| `traefik` | 18 | HTTPS reverse proxy on port 3000 |
| `selkies-fileserver` | 19 | Static file server for web UI |
| `relay-9222` | — | socat CDP port relay |
| `hermes` | autostart | Hermes AI agent process |
| `hermes-whatsapp` | manual | WhatsApp bridge (Node.js, disabled by default) |

## Ports

| Port | Service |
|------|---------|
| 3000 | Selkies web UI (Traefik HTTPS → static files + WebSocket proxy) |
| 9222 | Chrome DevTools Protocol (CDP, via socat relay) |

## Volumes

| Volume | Path | From |
|--------|------|------|
| `chrome-data` | `~/.chrome-debug` | selkies-desktop |
| `selkies-config` | `~/.config/selkies` | selkies-desktop |
| `data` | `/opt/data` | hermes |

## Access

Open `https://localhost:3000` in a browser. Accept the self-signed certificate warning (Traefik auto-generates a cert). The Selkies dashboard shows the labwc desktop with Chrome and Waybar.

HTTPS is required for WebCodecs API (`VideoDecoder`) used by the Selkies JS client.

## Quick Start

```bash
ov build selkies-desktop-hermes
ov config selkies-desktop-hermes -e OPENROUTER_API_KEY=sk-xxx
ov start selkies-desktop-hermes
# Access: https://localhost:3000 (accept cert warning)
ov wl screenshot selkies-desktop-hermes screenshot.png
```

## Configuration

The `hermes` layer's optional env vars are available at deploy time:

| Variable | Description |
|----------|-------------|
| `OPENROUTER_API_KEY` | API key for OpenRouter LLM inference |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token for messaging |
| `SLACK_BOT_TOKEN` | Slack bot token |
| `DISCORD_BOT_TOKEN` | Discord bot token |

```bash
ov config selkies-desktop-hermes -e OPENROUTER_API_KEY=sk-xxx -e TELEGRAM_BOT_TOKEN=xxx
```

The data volume (`/opt/data`) persists: sessions, skills, memories, logs, config, and `.env`.

## GPU Support

- **Rendering:** NVIDIA GPU via CDI (`DRINODE=/dev/dri/renderD129`), AMD/Intel via Mesa
- **Encoding:** NVENC detected but currently fails with driver 590.48. Falls back to CPU x264enc-striped at 60fps.

## Verification

```bash
ov status selkies-desktop-hermes              # All services RUNNING
curl -k https://localhost:3000               # 200 OK (Selkies HTML)
ov wl screenshot selkies-desktop-hermes t.png # Desktop screenshot via capture bridge
ov cdp status selkies-desktop-hermes          # CDP on port 9222
ov service status selkies-desktop-hermes      # hermes: RUNNING
ov shell selkies-desktop-hermes -c "hermes --version"
```

## Related Images

- `/ov-images:selkies-desktop` — desktop only, no Hermes agent
- `/ov-images:hermes` — Hermes agent only, no desktop
- `/ov-images:hermes-playwright` — Hermes agent with Playwright Chromium browser automation

## When to Use This Skill

**MUST be invoked** when the task involves the selkies-desktop-hermes image, deploying Hermes alongside a Selkies desktop, or comparing combined desktop+agent variants.
