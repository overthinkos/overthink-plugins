---
name: nvidia
description: |
  NVIDIA GPU runtime support: driver libs, nvidia-container-toolkit (CDI), and VA-API.
  Fedora (negativo17) and Arch Linux (pac). Base layer for all GPU-accelerated images.
  Use when working with NVIDIA GPU support, CDI device injection, or the nvidia layer.
---

# nvidia -- NVIDIA GPU runtime

Minimal NVIDIA runtime layer providing driver userspace libraries, `nvidia-container-toolkit` for CDI device injection, and VA-API hardware video acceleration. Supports both Fedora and Arch Linux.

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `root.yml` |
| Depends | none |

## Packages

**RPM** (from negativo17 fedora-multimedia repo):
- `nvidia-driver-libs` — NVIDIA GL/Vulkan userspace, `nvidia-smi`
- `nvidia-container-toolkit` — `nvidia-ctk` CLI for CDI spec generation
- `libva-nvidia-driver` — VA-API hardware video acceleration

**PAC** (Arch Linux):
- `nvidia-utils` — NVIDIA GL/Vulkan userspace, `nvidia-smi`
- `nvidia-container-toolkit` — `nvidia-ctk` CLI for CDI spec generation

## Environment Variables

| Variable | Value |
|----------|-------|
| `LD_LIBRARY_PATH` | `/usr/lib64` |

## CDI Support

The `nvidia-container-toolkit` provides `nvidia-ctk` which generates CDI (Container Device Interface) specs. `ov` calls `EnsureCDI()` before launching containers with GPU — if CDI specs don't exist at `/etc/cdi/nvidia.yaml`, it runs `nvidia-ctk cdi generate` to create them. This enables GPU access in nested containers where host CDI specs are not inherited.

## root.yml

Creates Vulkan ICD compatibility symlinks for nvidia-ctk CDI device injection.

## Used In Images

- `/ov-images:nvidia` — NVIDIA GPU base image (nvidia + cuda layers)
- `/ov-images:arch-ov` — Arch Linux ov development image (arch-ov + nvidia layers)

## Related Layers

- `/ov-layers:cuda` — CUDA development toolkit (depends on nvidia)
- `/ov-layers:rocm` — AMD GPU counterpart (ROCm runtime + OpenCL)
