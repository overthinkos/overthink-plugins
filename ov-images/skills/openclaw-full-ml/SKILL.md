---
name: openclaw-full-ml
description: |
  OpenClaw full deployment with ML tools (whisper, sherpa-onnx), Sway desktop, and Ollama.
  Use when working with the openclaw-full-ml image or GPU-accelerated OpenClaw deployments.
---

# openclaw-full-ml

Full OpenClaw with CUDA, Ollama for local LLMs, Whisper for speech-to-text, sherpa-onnx for offline TTS, Sway desktop, and all tool layers.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | openclaw-full-ml, sway-desktop, ollama |
| Platforms | linux/amd64 |
| Ports | 18789, 5900, 9222, 11434 |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway + Control UI | HTTP |
| 5900 | VNC (Sway desktop) | TCP |
| 9222 | Chrome DevTools Protocol | HTTP (relay) |
| 11434 | Ollama LLM server | HTTP |

## Quick Start

```bash
ov build openclaw-full-ml
ov start openclaw-full-ml
# Gateway at http://localhost:18789
# Ollama at http://localhost:11434
```

## ML Tools

- `whisper` -- OpenAI Whisper local speech-to-text (via pixi/conda-forge)
- `sherpa-onnx` -- Offline text-to-speech with Piper voices
- `ollama` -- Local LLM inference server

## Related Images

- `/ov-images:openclaw-full` -- same tools, no ML/GPU
- `/ov-images:openclaw-full-sway` -- same tools + desktop, no ML/GPU
- `/ov-images:openclaw-ollama-sway-browser` -- lighter variant with ollama but fewer tools

## When to Use This Skill

Use when the user asks about GPU-accelerated OpenClaw, local LLMs with OpenClaw, speech-to-text, or the openclaw-full-ml image.
