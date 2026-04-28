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
| Layers | agent-forwarding, openclaw-full-ml (metalayer), sway-desktop, ollama |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222, 9224, 11434 |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (wayvnc) | TCP |
| 9222 | Chrome DevTools | HTTP |
| 9224 | Chrome DevTools MCP (Streamable HTTP) | HTTP |
| 11434 | Ollama API | HTTP |

## Included Tools

Everything from `openclaw-sway-browser` plus:
- `ollama` — Local LLM inference server
- `whisper` — OpenAI Whisper local speech-to-text
- `sherpa-onnx` — Offline text-to-speech with Piper voices
- `cuda` — NVIDIA GPU acceleration

## Quick Start

```bash
ov image build openclaw-ollama-sway-browser
ov config openclaw-ollama-sway-browser
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
- `ov service status openclaw-ollama-sway-browser` — all services RUNNING
- `curl http://localhost:18789` — OpenClaw gateway responds
- `curl http://localhost:11434/api/tags` — Ollama API responds
- VNC client → `localhost:5900` — Sway desktop visible
- `curl http://localhost:9222/json` — Chrome DevTools responds
- `ov service status openclaw-ollama-sway-browser` should show `relay-18789` and `cdp-proxy` both RUNNING

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

**Prerequisites:** Chrome must be signed into Google with sync enabled (see Chrome Sign-In below). The Codex OAuth "Continue with Google" button requires an active Google session.

**Critical:** The `openclaw models auth login` TUI requires a real terminal. Running it piped through `tee` or with stdout redirected breaks the post-callback token exchange. **Use `ov tmux`** (see `/ov:tmux`):

```bash
IMG=openclaw-ollama-sway-browser

# 1. Start OAuth in a tmux session (provides real terminal)
ov tmux run $IMG -s oauth "openclaw models auth login --provider openai-codex --set-default"

# 2. Wait for URL, read it from tmux output
sleep 5
ov tmux capture $IMG -s oauth | grep -o 'https://auth.openai.com/[^ ]*'

# 3. Open the OAuth URL in Chrome (visible in VNC)
ov eval cdp open $IMG "<oauth-url>"

# 4. Click "Continue with Google" (VNC-visible via --vnc)
TAB=$(ov eval cdp list $IMG | grep -i "openai\|auth" | head -1 | awk '{print $1}')
ov eval cdp wait $IMG $TAB 'button._buttonStyleFix_wvuha_65' --timeout 15s
ov eval cdp click $IMG $TAB 'button._buttonStyleFix_wvuha_65' --vnc
sleep 5

# 5. Click "Continue" on Codex consent page (VNC-visible)
ov eval cdp click $IMG $TAB 'button._primary_3rdp0_107' --vnc

# 6. Verify token exchange completed
sleep 10
ov tmux capture $IMG -s oauth
# Should show: "OpenAI OAuth complete", "Default model set to openai-codex/gpt-5.4"

# 7. Restart gateway and verify
ov shell $IMG -c "supervisorctl restart openclaw"
sleep 5
ov shell $IMG -c "openclaw models status"
```

**Token persistence:** Auth tokens save to `~/.openclaw/agents/main/agent/auth-profiles.json` in the `data` volume. Survives `ov stop`/`ov start` and image rebuilds. Only destroyed by `ov remove --purge`.

**Why `ov tmux`:** The `openclaw models auth login` TUI needs an active terminal to process the OAuth callback. The callback server (port 1455) receives the authorization code and displays "Authentication successful" in the browser, but the token exchange with OpenAI's servers requires the TUI event loop to be responsive. `ov shell --tty` uses `script(1)` which breaks when piped/backgrounded. `ov tmux run` provides a real tmux terminal. See `/ov:tmux` for full documentation.

**Stale port 1455:** If a previous OAuth attempt left port 1455 occupied, kill it: `ov shell $IMG -c 'kill -9 $(ss -tlnp sport = :1455 | grep -oP "pid=\K\d+")'`

**Callback architecture:** The OAuth redirect to `http://127.0.0.1:1455/auth/callback` is container-internal — Chrome and the callback server share the same network namespace. No port mapping needed for 1455.

See `/ov:openclaw` for full gateway configuration reference.

## Chrome Sign-In and Sync

**MANDATORY:** Full Chrome profile setup with Google sign-in and sync enabled. Never skip steps or use "Chrome without an account". All interactions must be VNC-visible.

**Required tools:**
- `ov eval cdp click --vnc` — finds element via CDP (shadow DOM aware), clicks via VNC (visible)
- `ov eval cdp coords` — shows element position in viewport and desktop coordinates
- `ov eval vnc mouse` — moves cursor to verify position before clicking native UI
- `ov eval vnc click` — clicks Chrome native UI elements (infobubbles, dialogs not in DOM)
- `ov eval vnc screenshot` — verifies state after every action

**Critical shm_size:** The `shm_size: "1g"` in chrome layer.yml is essential. Without it, Chrome crashes with `ContextResult::kTransientFailure` on complex pages (Google sign-in, OpenAI auth). If Chrome crashes on page load, check: `ov shell $IMG -c 'df -h /dev/shm'` — must show 1.0G, not 63M. Fix: `ov remove $IMG && ov config $IMG && ov start $IMG` with updated `ov` binary.

**DO NOT** open `chrome://intro` on NVIDIA+pixman setups — it crashes Chrome due to GPU-intensive animations. Use `https://accounts.google.com/signin` directly.

```bash
IMG=openclaw-ollama-sway-browser

# Prerequisites: GMAIL_USER and GMAIL_PASSWORD in env (App Password for 2FA)
# Wait 30s after container start for Chrome + VNC to stabilize

# Step 1: Google sign-in via accounts.google.com (NOT chrome://intro)
ov eval cdp open $IMG "https://accounts.google.com/signin"
sleep 5
TAB=$(ov eval cdp list $IMG | grep -i "sign in\|accounts" | head -1 | awk '{print $1}')

# Step 2: Enter email (VNC-visible clicks)
ov eval cdp click $IMG $TAB 'input[type="email"]' --vnc && sleep 1
ov eval cdp type $IMG $TAB 'input[type="email"]' "$GMAIL_USER"
ov eval cdp click $IMG $TAB '#identifierNext' --vnc && sleep 5

# Step 3: Enter password
ov eval cdp wait $IMG $TAB 'input[type="password"]' --timeout 15s
ov eval cdp click $IMG $TAB 'input[type="password"]' --vnc && sleep 1
ov eval cdp type $IMG $TAB 'input[type="password"]' "$GMAIL_PASSWORD"
ov eval cdp click $IMG $TAB '#passwordNext' --vnc && sleep 7

# Step 4: Dismiss Chrome first-run dialogs (appear after Google sign-in)
# 4a: Search engine choice (EU DMA) — select Google, click Set as default
#     Uses shadow DOM: cr-radio-button elements, #actionButton
SE_TAB=$(ov eval cdp list $IMG | grep "search-engine" | head -1 | awk '{print $1}')
ov eval cdp click $IMG $SE_TAB 'cr-radio-button:nth-of-type(6)' --vnc   # Google (6th in randomized list — verify with eval)
sleep 1
ov eval cdp click $IMG $SE_TAB '#actionButton' --vnc                      # "Set as default"
sleep 2

# 4b: "Make Chrome your own" — click "Continue as <user>" for browser sync
#     This is a chrome:// DICE intercept tab with shadow DOM
DICE_TAB=$(ov eval cdp list $IMG | grep "signin-dice" | head -1 | awk '{print $1}')
ov eval cdp click $IMG $DICE_TAB '#accept-button' --vnc                   # "Continue as <user>"
sleep 3

# Step 5: Enable sync — "Turn on Sync" then "Yes, I'm in"
ov eval cdp open $IMG "chrome://settings/syncSetup"
sleep 3
SYNC_TAB=$(ov eval cdp list $IMG | grep "settings" | head -1 | awk '{print $1}')
ov eval cdp click $IMG $SYNC_TAB '#sync-button' --vnc                     # "Turn on Sync"
sleep 2
# "Yes, I'm in" is Chrome native UI — use ov eval vnc mouse to find, then ov eval vnc click
ov eval vnc mouse $IMG 1130 565       # Verify cursor is on "Yes, I'm in"
ov eval vnc screenshot $IMG /tmp/verify-yesin.png  # Check position
ov eval vnc click $IMG 1130 565       # Click "Yes, I'm in"
sleep 3

# Step 6: Dismiss "Restore pages?" if present (Chrome native UI)
# Use ov eval vnc mouse to find X button, then click
# The X is at approximately VNC (1880, 117) — verify with ov eval vnc mouse first
ov eval vnc mouse $IMG 1880 117 && ov eval vnc screenshot $IMG /tmp/verify-restore-x.png
ov eval vnc click $IMG 1880 117

# Step 7: Verify full sign-in and sync
ov shell $IMG -c 'python3 << "PYEOF"
import json, os
with open(os.path.expanduser("~/.chrome-debug/Local State")) as f:
    d = json.load(f)
info = d.get("profile",{}).get("info_cache",{}).get("Default",{})
print("gaia_id:", info.get("gaia_id",""))
print("signed_in:", info.get("is_consented_primary_account", False))
print("user_name:", info.get("user_name",""))
PYEOF'
# Expected: gaia_id populated, signed_in: True, user_name: <email>
```

**Coordinate discovery for Chrome native UI:** VNC coordinates from screenshots don't match pixel positions in the displayed image (the image viewer scales). Always use `ov eval vnc mouse <x> <y>` + `ov eval vnc screenshot` to verify the cursor position before clicking. For web page elements, `ov eval cdp click --vnc` handles coordinate translation automatically.

**Search engine list order:** The EU DMA search engine choice presents engines in randomized order. The `cr-radio-button:nth-of-type(6)` selector may not always be Google. Verify with: `ov eval cdp eval $IMG $TAB '...'` to find which nth-of-type index has "Google" in its text.

**VNC startup delay:** VNC may show a blank screen for ~10-15s after container start while the DPMS workaround triggers wayvnc's capture pipeline. CDP screenshots work immediately. Wait for VNC to stabilize before relying on VNC screenshots for verification.

## Data Persistence

All persistent state lives in three named volumes:

| Volume Name | Container Path | Contents |
|-------------|----------------|----------|
| `ov-...-data` | `~/.openclaw` | Config, auth tokens, sessions, models.json |
| `ov-...-chrome-data` | `~/.chrome-debug` | Chrome profile, Google sign-in, sync, cookies |
| `ov-...-models` | `~/.ollama` | Downloaded LLM models |

| Operation | Volumes |
|-----------|---------|
| `ov stop` + `ov start` | Preserved |
| `ov remove` (no `--purge`) | Preserved |
| `ov image build` (image rebuild) | Preserved |
| `ov remove --purge` | **DESTROYED** |

Volume metadata is stored as OCI labels (`org.overthinkos.volumes`) in the image. `ov config` always recreates the correct mounts even after image rebuild.

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-ollama-sway-browser image, GPU-accelerated OpenClaw, or the full-stack AI desktop. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
