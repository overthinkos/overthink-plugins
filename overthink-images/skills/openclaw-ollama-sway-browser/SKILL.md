---
name: openclaw-ollama-sway-browser
description: |
  Full-stack AI image: OpenClaw gateway + Ollama LLM server + Sway desktop
  with Chrome. GPU-accelerated with CUDA. Use when working with the
  combined openclaw+ollama desktop deployment.
---

# openclaw-ollama-sway-browser

Full-stack AI deployment: OpenClaw gateway, local LLM inference via Ollama, and a Wayland desktop with Chrome.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | openclaw, sway-desktop, ollama |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222, 11434 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `nvidia` (CUDA base)
2. `pixi` → `python` → `supervisord` (transitive)
3. `nodejs` (transitive)
4. `openclaw` — gateway on :18789
5. `dbus` → `sway` (transitive via sway-desktop)
6. `pipewire`, `wayvnc`, `chrome-sway`, `xfce4-terminal`, `thunar`, `waybar`
7. `ollama` — LLM server on :11434

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (wayvnc) | TCP |
| 9222 | Chrome DevTools | HTTP |
| 11434 | Ollama API | HTTP |

## Quick Start

```bash
ov build openclaw-ollama-sway-browser
ov start openclaw-ollama-sway-browser
# Pull a model: ov shell openclaw-ollama-sway-browser -c "ollama pull llama3"
```

## Key Layers

- `/overthink-layers:openclaw` — AI gateway service
- `/overthink-layers:ollama` — LLM inference server
- `/overthink-layers:sway-desktop` — full desktop composition
- `/overthink-layers:cuda` — GPU support (via nvidia base)

## Related Images

- `/overthink-images:openclaw-sway-browser` — same but without Ollama (no GPU needed)
- `/overthink-images:openclaw-ollama` — headless OpenClaw + Ollama (no desktop)
- `/overthink-images:nvidia` — GPU base

## When to Use This Skill

Use when the user asks about the combined openclaw+ollama+desktop image, self-hosted AI with local models and browser automation, or the full-stack GPU deployment.
