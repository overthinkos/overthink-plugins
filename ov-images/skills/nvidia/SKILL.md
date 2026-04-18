---
name: nvidia
description: |
  NVIDIA GPU base image with runtime support and CUDA toolkit on Fedora. Base for all
  GPU-accelerated images (python-ml, jupyter, ollama, comfyui).
  MUST be invoked before building, deploying, configuring, or troubleshooting the nvidia image.
---

# nvidia

NVIDIA GPU base image — provides GPU runtime (driver libs, CDI toolkit) and CUDA development toolkit on Fedora.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora-nonfree (fedora + RPM Fusion) |
| Layers | agent-forwarding, nvidia, cuda |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora-nonfree` (fedora + rpmfusion — provides RPM Fusion free + nonfree repos)
2. `nvidia` — GPU runtime: driver libs, nvidia-container-toolkit (CDI), VA-API
3. `cuda` — CUDA toolkit, cuDNN, onnxruntime, ffmpeg-libs

## Quick Start

```bash
ov image build nvidia
ov shell nvidia
```

## Key Layers

- `/ov-layers:nvidia` — NVIDIA GPU runtime (driver libs, CDI generation via nvidia-ctk)
- `/ov-layers:cuda` — CUDA toolkit and GPU development libraries

## Derived Images

- `/ov-images:python-ml` — ML Python environment
- `/ov-images:jupyter` — Jupyter notebook server
- `/ov-images:ollama` — LLM inference server
- `/ov-images:comfyui` — image generation UI
- `/ov-images:openclaw-ollama` — OpenClaw + Ollama
- `/ov-images:openclaw-ollama-sway-browser` — OpenClaw + Ollama + desktop

## Related Images

- `/ov-images:fedora` — parent base (no GPU)
- `/ov-images:arch-ov` — Arch Linux ov toolchain with nvidia (shared layers)
- `/ov-images:fedora-ov` — Fedora ov toolchain with nvidia (shared layers)

## Verification

After `ov image build`:
- `ov shell nvidia -c "nvidia-smi"` — GPU info
- `ov shell nvidia -c "nvidia-ctk --version"` — CDI toolkit
- `ov shell nvidia -c "nvcc --version"` — CUDA compiler

## When to Use This Skill

**MUST be invoked** when the task involves the nvidia image, GPU base images, CUDA in containers, or CDI device configuration. Invoke this skill BEFORE reading source code or launching Explore agents.
