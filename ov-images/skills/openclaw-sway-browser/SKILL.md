---
name: openclaw-sway-browser
description: |
  Maximal OpenClaw deployment with Sway desktop, Chrome, VNC, and all tool layers.
  Includes all feasible OpenClaw skill dependencies. Use when working with
  MUST be invoked before building, deploying, configuring, or troubleshooting the openclaw-sway-browser image.
---

# openclaw-sway-browser

Maximal OpenClaw gateway with full Wayland desktop, Chrome browser, VNC access, and all tool layers for maximum skill coverage.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, openclaw-full (metalayer: 28 layers), sway-desktop |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222, 9224 |
| Tunnel | tailscale (all ports) |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (wayvnc) | TCP |
| 9222 | Chrome DevTools | HTTP |
| 9224 | Chrome DevTools MCP (Streamable HTTP) | HTTP |

## Included Tools

npm: codex, gemini, clawhub, mcporter, oracle, xurl, summarize, playwright, claude-code
Go: blogwatcher, gifgrep, wacli, goplaces, songsee, sag, camsnap, gogcli, ordercli
Cargo: himalaya
Python: uv, nano-pdf
RPM: gh, git, tmux, ffmpeg, ripgrep, sqlite

## Quick Start

```bash
ov image build openclaw-sway-browser
ov config openclaw-sway-browser
ov start openclaw-sway-browser
# Gateway at http://localhost:18789
# VNC desktop at localhost:5900
```

## Key Layers

- `/ov-layers:openclaw-full` — metalayer composing openclaw + chrome + 26 tool layers
- `/ov-layers:sway-desktop` — full desktop (pipewire, wayvnc, chrome-sway, terminal, file manager, waybar)

## Related Images

- `/ov-images:openclaw` — headless gateway only (minimal)
- `/ov-images:openclaw-ollama-sway-browser` — adds CUDA, Ollama, Whisper, sherpa-onnx for ML

## Verification

After `ov start`:
- `ov status openclaw-sway-browser` — container running
- `ov service status openclaw-sway-browser` — all services RUNNING
- `curl -s http://localhost:18789` — OpenClaw gateway responds
- VNC client connects to `localhost:5900` — desktop accessible
- `curl -s http://localhost:9222/json/version` — Chrome DevTools responds

## Port Relay Architecture

OpenClaw (18789) and Chrome DevTools (9222) both use port relay (socat) — services bind to loopback, socat forwards from the container interface. This avoids origin/security checks that block non-loopback connections.

## Gateway First-Run Configuration

After the first container start, the gateway needs one-time setup before it will accept requests:

```bash
IMG=openclaw-sway-browser

# Required: gateway mode (without this, gateway refuses to start)
ov shell $IMG -c "openclaw config set gateway.mode local"

# Required: Chrome CDP integration
ov shell $IMG -c "openclaw config set browser.cdpUrl 'http://127.0.0.1:9222'"

# Apply changes
ov shell $IMG -c "supervisorctl restart openclaw"
```

`dangerouslyAllowHostHeaderOriginFallback` is **NOT needed** -- the gateway binds to loopback only, and port_relay (socat) handles external access.

### OpenAI Codex OAuth

**Prerequisites:** Chrome must be signed into Google with sync enabled. See `/ov-images:openclaw-ollama-sway-browser` for the full Chrome sign-in and Codex OAuth procedure — the process is identical for this image (substitute `IMG=openclaw-sway-browser`).

**Key points:**
- Use `ov tmux run` for the OAuth TUI (not `--tty` piped through `tee`) — see `/ov:tmux`
- Click "Continue with Google" then "Continue" on consent using `ov test cdp click --vnc`
- Callback at `localhost:1455` is container-internal (no port mapping needed)
- Model: `openai-codex/gpt-5.4`
- Tokens persist in the `data` volume (`~/.openclaw`)

See `/ov:openclaw` for full gateway configuration reference.

## Data Persistence

| Volume | Container Path | Contents |
|--------|----------------|----------|
| `ov-...-data` | `~/.openclaw` | Config, auth tokens, sessions |
| `ov-...-chrome-data` | `~/.chrome-debug` | Chrome profile, sign-in, sync |

Volumes survive `ov stop`/`ov start` and image rebuilds. Only destroyed by `ov remove --purge`.

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-sway-browser image, OpenClaw with browser automation, or desktop gateway deployments. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
