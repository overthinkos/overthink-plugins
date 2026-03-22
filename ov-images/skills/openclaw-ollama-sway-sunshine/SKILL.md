---
name: openclaw-ollama-sway-sunshine
description: |
  Full-stack AI image with Sunshine game streaming, OpenClaw, Ollama, and CUDA GPU.
  Use when working with GPU-accelerated OpenClaw desktop with Sunshine streaming.
---

# openclaw-ollama-sway-sunshine

Full-stack GPU-accelerated AI deployment: OpenClaw gateway with all tool layers, local LLM inference via Ollama, Whisper STT, sherpa-onnx TTS, and a Sway desktop with Chrome — streamed via Sunshine/Moonlight instead of VNC.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia (fedora + CUDA) |
| Layers | openclaw-full-ml (metalayer), sway-desktop-sunshine, ollama |
| Platforms | linux/amd64 |
| Ports | 18789, 9222, 11434, 47984, 47989, 47990, 48010, 47998/udp, 47999/udp, 48000/udp |
| Registry | ghcr.io/overthinkos |

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway | HTTP |
| 9222 | Chrome DevTools | HTTP |
| 11434 | Ollama API | HTTP |
| 47990 | Sunshine Web UI | HTTPS |
| 47998 | Video stream | UDP |
| 47999 | Control channel | UDP |
| 48000 | Audio stream | UDP |

## Quick Start

```bash
ov build openclaw-ollama-sway-sunshine
ov start openclaw-ollama-sway-sunshine
# Wait for services to start (~20s)
ov sun passwd openclaw-ollama-sway-sunshine --generate   # Set Sunshine Web UI credentials
ov moon pair openclaw-ollama-sway-sunshine --auto        # Pair as Moonlight client
ov sun status openclaw-ollama-sway-sunshine               # Verify health
# Gateway at http://localhost:18789
# Ollama at http://localhost:11434
# Sunshine at https://localhost:47990
```

## Key Difference from VNC Variant

This image uses Sunshine instead of wayvnc, enabling the full GPU pipeline on NVIDIA headless:
- Sway renders with gles2 (GPU) instead of pixman (CPU)
- Chrome has full GPU acceleration (VAAPI, NVIDIA EGL)
- Sunshine encodes with NVENC (H.264/HEVC/AV1)
- No gray/blank screens (the wayvnc upstream bug doesn't apply)

## Key Layers

- `/ov-layers:openclaw-full-ml` -- metalayer: openclaw-full + whisper + sherpa-onnx
- `/ov-layers:ollama` -- LLM inference server
- `/ov-layers:sway-desktop-sunshine` -- GPU-accelerated desktop with Sunshine
- `/ov-layers:cuda` -- GPU support (via nvidia base)

## Related Images

- `/ov-images:openclaw-ollama-sway-browser` -- same stack but with VNC (wayvnc)
- `/ov-images:sway-browser-sunshine` -- lightweight Sunshine desktop (no OpenClaw/Ollama)

## When to Use This Skill

Use when the user asks about GPU-accelerated OpenClaw with Sunshine, the full-stack AI desktop with game streaming, or comparing this to the VNC variant.
