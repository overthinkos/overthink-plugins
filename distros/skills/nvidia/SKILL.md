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
`/ov-distros:cachyos`.

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
ov box build nvidia
ov shell nvidia
```

## Key Layers

- `/ov-distros:nvidia` — NVIDIA GPU runtime (driver libs, CDI generation via nvidia-ctk)
- `/ov-distros:cuda` — CUDA toolkit and GPU development libraries

## Derived Images

- `/ov-languages:python-ml` — ML Python environment
- `/ov-jupyter:jupyter` — Jupyter notebook server
- `/ov-ollama:ollama` — LLM inference server
- `/ov-comfyui:comfyui` — image generation UI

## Related Images

- `/ov-distros:fedora` — parent base (no GPU)
- `/ov-distros:cachyos` — `cachyos.nvidia` is the CachyOS GPU-base sibling (Arch/CachyOS GPU stack)
- `/ov-coder:arch-ov` — Arch Linux ov toolchain with nvidia (shared layers)
- `/ov-distros:fedora-ov` — Fedora ov toolchain with nvidia (shared layers)

## Verification

After `ov box build`:
- `ov shell nvidia -c "nvidia-smi"` — GPU info
- `ov shell nvidia -c "nvidia-ctk --version"` — CDI toolkit
- `ov shell nvidia -c "nvcc --version"` — CUDA compiler

## When to Use This Skill

**MUST be invoked** when the task involves the nvidia image, GPU base images, CUDA in containers, or CDI device configuration. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
