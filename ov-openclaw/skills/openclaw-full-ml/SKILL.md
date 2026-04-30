---
name: openclaw-full-ml
description: |
  OpenClaw full + ML tools + Ollama + Sway desktop + VNC. GPU-accelerated.
  Currently disabled. Enable in image.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the openclaw-full-ml image.
---

# openclaw-full-ml

GPU-accelerated OpenClaw with all tools, ML capabilities (whisper, sherpa-onnx TTS), Ollama, Sway desktop, and VNC.

## Image Properties

| Property | Value |
|----------|-------|
| Base | nvidia |
| Layers | agent-forwarding, openclaw-full-ml, sway-desktop-vnc, ollama |
| Platforms | linux/amd64 |
| Ports | 18789 (gateway), 5900 (VNC), 9222 (CDP), 11434 (Ollama) |
| Status | **disabled** (set `enabled: true` in image.yml) |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` → `fedora-nonfree` → `nvidia` (CUDA base)
2. `openclaw-full-ml` metalayer:
   - `openclaw-full` (28 tool layers)
   - `whisper` — speech-to-text
   - `sherpa-onnx` — offline TTS
3. `sway-desktop-vnc` — desktop + VNC
4. `ollama` — LLM inference server

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway | HTTP |
| 5900 | VNC | TCP |
| 9222 | Chrome DevTools | HTTP |
| 11434 | Ollama API | HTTP |

## Quick Start

```bash
# Enable in image.yml first (remove enabled: false)
ov image build openclaw-full-ml
ov config openclaw-full-ml
ov start openclaw-full-ml
```

## Key Layers

- `/ov-openclaw:openclaw-full-ml` — Full tools + ML (whisper, sherpa-onnx)
- `/ov-ollama:ollama` — LLM server
- `/ov-foundation:cuda` — GPU support (via nvidia base)

## Related Images

- `/ov-openclaw:openclaw-ollama-sway-browser` — enabled GPU variant (same concept)
- `/ov-openclaw:openclaw-full-sway` — without ML/Ollama (disabled)
- `/ov-openclaw:openclaw-full` — headless, no ML (disabled)

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-full-ml image or the combined ML-capable OpenClaw desktop deployment.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
