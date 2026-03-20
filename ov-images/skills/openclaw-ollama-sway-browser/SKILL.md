---
name: openclaw-ollama-sway-browser
description: |
  Full-stack AI image: OpenClaw gateway + all tools + Ollama LLM + Whisper STT +
  sherpa-onnx TTS + Sway desktop with Chrome. GPU-accelerated with CUDA.
  Use when working with the combined ML-capable OpenClaw desktop deployment.
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

## When to Use This Skill

Use when the user asks about GPU-accelerated OpenClaw, local LLMs with OpenClaw, speech-to-text, the full-stack AI desktop, or the openclaw-ollama-sway-browser image.
