---
name: nvidia
description: |
  NVIDIA GPU base image with CUDA toolkit on Fedora. Base for all
  GPU-accelerated images (python-ml, jupyter, ollama, comfyui).
  MUST be invoked before building, deploying, configuring, or troubleshooting the nvidia image.
---

# nvidia

NVIDIA GPU base image — provides CUDA toolkit and GPU libraries on Fedora.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora-nonfree (fedora + RPM Fusion) |
| Layers | cuda |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora-nonfree` (fedora + rpmfusion — provides RPM Fusion free + nonfree repos)
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

## Verification

After `ov build`:
- `ov list` — image appears in list
- `ov shell nvidia` — interactive shell works

## When to Use This Skill

**MUST be invoked** when the task involves the nvidia image, GPU base images, or CUDA in containers. Invoke this skill BEFORE reading source code or launching Explore agents.
