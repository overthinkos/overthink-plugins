---
name: cuda
description: |
  CUDA toolkit, cuDNN, ONNX Runtime, and NVIDIA GPU development libraries from negativo17 repos.
  Depends on the nvidia layer for runtime support. Use when working with GPU computing, CUDA,
  cuDNN, machine learning infrastructure, or NVIDIA development tools.
---

# cuda -- CUDA development toolkit

CUDA compiler, cuDNN, ONNX Runtime, and GPU development libraries. Depends on the `nvidia` layer for runtime support (driver libs, nvidia-ctk).

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `root.yml`, `layer.yml` |
| Depends | `nvidia` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CUDA_HOME` | `/usr` |

## Packages

RPM (from negativo17 fedora-multimedia repo): `cuda-nvcc`, `cuda-cudart-devel`, `cuda-cudart-static`, `cuda-nvrtc-devel`, `cuda-cupti-devel`, `cuda-cccl-devel`, `cuda-cudnn`, `libcurand-devel`, `libcufile-devel`, `onnxruntime`, `ffmpeg-libs`, `libaio-devel`, `cpio`

**Note:** No pac section — CUDA development is Fedora-only. The `nvidia` layer provides Arch Linux GPU runtime support.

## root.yml

Extracts cuDNN headers from `cuda-cudnn-devel` RPM (bypasses driver dependency via rpm2cpio).

## Usage

```yaml
# images.yml
nvidia:
  base: fedora-nonfree
  layers:
    - nvidia
    - cuda
```

## Used In Images

- `/ov-images:nvidia` (base for all GPU images)
- `/ov-images:immich-cuda`

## Related Layers

- `/ov-layers:nvidia` — NVIDIA GPU runtime (driver libs, CDI toolkit) — required dependency
- `/ov-layers:rocm` — AMD GPU counterpart (ROCm runtime + OpenCL)
- `/ov-layers:python-ml` — ML Python environment (depends on cuda)
- `/ov-layers:jupyter` — Jupyter notebooks (depends on cuda)
- `/ov-layers:ollama` — LLM server (depends on cuda)
- `/ov-layers:comfyui` — image generation (depends on cuda)
