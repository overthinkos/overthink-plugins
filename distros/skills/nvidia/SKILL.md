---
name: nvidia
description: |
  NVIDIA GPU base image with runtime support and CUDA toolkit on Fedora. Base for all
  GPU-accelerated images (python-ml, jupyter, ollama, comfyui).
  MUST be invoked before building, deploying, configuring, or troubleshooting the nvidia image.
---

# nvidia

NVIDIA GPU base image — provides GPU runtime (driver libs, CDI toolkit) and CUDA development toolkit on Fedora.

The `nvidia` and `cuda` layers are multi-distro (Fedora rpm + Arch pac), so a
CachyOS GPU base exists alongside this Fedora `nvidia` image: `cachyos.nvidia`
(cachyos + agent-forwarding + nvidia + cuda) in the `overthinkos/cachyos`
submodule. The two are siblings — pick the Fedora `nvidia` image for the
RPM-based GPU stack, `cachyos.nvidia` for the Arch/CachyOS GPU stack. See
`/charly-distros:cachyos`.

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

The `nvidia` image lives in the `overthinkos/fedora` submodule (`image/fedora`):

```bash
charly -C image/fedora image build nvidia
charly shell nvidia
```

## Key Layers

- `/charly-distros:nvidia` — NVIDIA GPU runtime (driver libs, CDI generation via nvidia-ctk)
- `/charly-distros:cuda` — CUDA toolkit and GPU development libraries

## Derived Images

- `/charly-languages:python-ml` — ML Python environment
- `/charly-jupyter:jupyter` — Jupyter notebook server
- `/charly-ollama:ollama` — LLM inference server
- `/charly-comfyui:comfyui` — image generation UI

## Related Images

- `/charly-distros:fedora` — parent base (no GPU)
- `/charly-distros:cachyos` — `cachyos.nvidia` is the CachyOS GPU-base sibling (Arch/CachyOS GPU stack)
- `/charly-coder:charly-arch` — Arch Linux charly toolchain with nvidia (shared layers)
- `/charly-distros:charly-fedora` — Fedora charly toolchain with nvidia (shared layers)

## Verification

After `charly box build`:
- `charly shell nvidia -c "nvidia-smi"` — GPU info
- `charly shell nvidia -c "nvidia-ctk --version"` — CDI toolkit
- `charly shell nvidia -c "nvcc --version"` — CUDA compiler

## When to Use This Skill

**MUST be invoked** when the task involves the nvidia image, GPU base images, CUDA in containers, or CDI device configuration. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
