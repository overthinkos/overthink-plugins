---
name: openclaw-ollama-sway-browser
description: |
  Full-stack AI image: OpenClaw gateway + all tools + Ollama LLM + Whisper STT +
  sherpa-onnx TTS + Sway desktop with Chrome. GPU-accelerated with CUDA.
  MUST be invoked before building, deploying, configuring, or troubleshooting the openclaw-ollama-sway-browser image.
---

# openclaw-ollama-sway-browser

Full-stack GPU-accelerated AI deployment: OpenClaw gateway with all tool layers, local LLM inference via Ollama, Whisper speech-to-text, sherpa-onnx offline TTS, and a Wayland desktop with Chrome.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | openclaw-full-ml (metalayer), sway-desktop, ollama |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222, 11434 |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (wayvnc) | TCP |
| 9222 | Chrome DevTools | HTTP |
| 11434 | Ollama API | HTTP |

## Included Tools

Everything from `openclaw-sway-browser` plus:
- `ollama` — Local LLM inference server
- `whisper` — OpenAI Whisper local speech-to-text
- `sherpa-onnx` — Offline text-to-speech with Piper voices
- `cuda` — NVIDIA GPU acceleration

## Quick Start

```bash
ov build openclaw-ollama-sway-browser
ov enable openclaw-ollama-sway-browser
ov start openclaw-ollama-sway-browser
# Gateway at http://localhost:18789
# Ollama at http://localhost:11434
# Pull a model: ov shell openclaw-ollama-sway-browser -c "ollama pull llama3"
```

## Key Layers

- `/ov-layers:openclaw-full-ml` — metalayer: openclaw-full + whisper + sherpa-onnx
- `/ov-layers:ollama` — LLM inference server
- `/ov-layers:sway-desktop` — full desktop composition
- `/ov-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-images:openclaw-sway-browser` — same tools but no GPU/ML (lighter)
- `/ov-images:openclaw-ollama` — headless OpenClaw + Ollama (no desktop, no tools)
- `/ov-images:openclaw` — minimal headless gateway

## Verification

After `ov start`:
- `ov status openclaw-ollama-sway-browser` — container running
- `ov service status openclaw-ollama-sway-browser` — all supervisord services RUNNING
- `curl http://localhost:18789` — OpenClaw gateway responds
- `curl http://localhost:11434/api/tags` — Ollama API responds
- VNC client → `localhost:5900` — Sway desktop visible
- `curl http://localhost:9222/json` — Chrome DevTools responds
- `ov service status openclaw-ollama-sway-browser` should show `relay-18789` and `relay-9222` both RUNNING

## Port Relay Architecture

OpenClaw (18789) and Chrome DevTools (9222) both use port relay (socat) — services bind to loopback, socat forwards from the container interface. This avoids origin/security checks that block non-loopback connections.

## Gateway First-Run Configuration

After the first container start, the gateway needs one-time setup before it will accept requests:

```bash
IMG=openclaw-ollama-sway-browser

# Required: gateway mode (without this, gateway refuses to start)
ov shell $IMG -c "openclaw config set gateway.mode local"

# Required: Chrome CDP integration
ov shell $IMG -c "openclaw config set browser.cdpUrl 'http://127.0.0.1:9222'"

# Apply changes
ov shell $IMG -c "supervisorctl restart openclaw"
```

`dangerouslyAllowHostHeaderOriginFallback` is **NOT needed** — the gateway binds to loopback only, and port_relay (socat) handles external access.

### OpenAI Codex OAuth

```bash
# Interactive OAuth (--tty required for PTY)
ov shell $IMG --tty -c "openclaw models auth login --provider openai-codex --set-default"
```

The `BROWSER=browser-open` env var auto-opens the OAuth URL in Chrome via CDP. The callback at `http://127.0.0.1:1455/auth/callback` is container-internal (no port mapping needed). Model: `openai-codex/gpt-5.4`.

See `/ov:openclaw` for full gateway configuration reference.

## Chrome Sign-In

Sign into Chrome with Gmail credentials for sync. Requires `GMAIL_USER` and `GMAIL_PASSWORD` in `.env` (App Password for 2FA accounts). Both ports 9222 (CDP) and 5900 (VNC) are needed for the hybrid automation pattern.

```bash
IMG=openclaw-ollama-sway-browser

# Fresh start (removes existing profile)
ov remove $IMG --purge
ov enable $IMG && ov start $IMG
sleep 60  # wait for Chrome + VNC to stabilize

# Dismiss first-run dialog
ov sway msg $IMG 'focus left' && ov vnc key $IMG Return

# Click "Sign in" on chrome://intro (--vnc required for chrome:// pages)
TAB=$(ov cdp list $IMG | grep intro | head -1 | awk '{print $1}')
ov cdp click $IMG $TAB '#acceptSignInButton' --vnc
sleep 3
TAB=$(ov cdp list $IMG | grep -i "sign in" | head -1 | awk '{print $1}')

# Enter email
ov cdp wait $IMG $TAB '#identifierId' --timeout 30s
ov cdp click $IMG $TAB '#identifierId' --vnc && sleep 0.5
ov vnc type $IMG "$GMAIL_USER"
ov cdp click $IMG $TAB '#identifierNext' --vnc && sleep 5

# Enter password
ov cdp wait $IMG $TAB 'input[type="password"]' --timeout 15s
ov cdp click $IMG $TAB 'input[type="password"]' --vnc && sleep 0.5
ov vnc type $IMG "$GMAIL_PASSWORD"
ov cdp click $IMG $TAB '#passwordNext' --vnc && sleep 7

# Enable sync
ov cdp click $IMG $TAB '#confirmButton' --vnc
```

See `/ov:cdp` for the full recipe with verification checkpoints and challenge handling.

**VNC startup delay:** VNC shows a blank gray screen for ~30-70s after container start due to a transient wayvnc race condition. CDP screenshots work immediately. Wait for VNC to stabilize before relying on VNC screenshots for verification.

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-ollama-sway-browser image, GPU-accelerated OpenClaw, or the full-stack AI desktop. Invoke this skill BEFORE reading source code or launching Explore agents.
