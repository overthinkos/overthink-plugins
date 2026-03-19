---
name: nvidia
description: |
  NVIDIA GPU base image with CUDA toolkit on Fedora. Base for all
  GPU-accelerated images (python-ml, jupyter, ollama, comfyui).
  Use when working with GPU images or CUDA support.
---

# nvidia

NVIDIA GPU base image — provides CUDA toolkit and GPU libraries on Fedora.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | cuda |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `cuda` — CUDA toolkit, cuDNN, onnxruntime, ffmpeg-libs

## Quick Start

```bash
ov build nvidia
ov shell nvidia
```

## Key Layers

- `/ov-layers:cuda` — CUDA toolkit and GPU libraries

## Derived Images

- `/ov-images:python-ml` — ML Python environment
- `/ov-images:jupyter` — Jupyter notebook server
- `/ov-images:ollama` — LLM inference server
- `/ov-images:comfyui` — image generation UI
- `/ov-images:openclaw-ollama` — OpenClaw + Ollama
- `/ov-images:openclaw-ollama-sway-browser` — OpenClaw + Ollama + desktop

## Related Images

- `/ov-images:fedora` — parent base (no GPU)

## When to Use This Skill

Use when the user asks about the nvidia image, GPU base images, CUDA in containers, or which images have GPU support.
