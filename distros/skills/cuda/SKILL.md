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
| Install files | `task:`, `layer.yml` |
| Depends | `nvidia`, `ffmpeg` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CUDA_HOME` | `/usr` |

## Packages

RPM: `cuda-nvcc`, `cuda-cudart-devel`, `cuda-cudart-static`, `cuda-nvrtc-devel`, `cuda-cupti-devel`, `cuda-cccl-devel`, `cuda-cudnn`, `libcurand-devel`, `libcufile-devel`, `onnxruntime`, `libaio-devel`, `cpio`

**Note:** FFmpeg codec libraries are provided via the `ffmpeg` dependency layer rather than installed directly. CUDA packages come from the negativo17 `fedora-nvidia` repo (added by the `nvidia` layer).

**Note:** No pac section — CUDA development is Fedora-only. The `nvidia` layer provides Arch Linux GPU runtime support.

## Install tasks

Extracts cuDNN headers from `cuda-cudnn-devel` RPM (bypasses driver dependency via rpm2cpio).

## Usage

```yaml
# image.yml
nvidia:
  base: fedora-nonfree
  layers:
    - nvidia
    - cuda
```

## Used In Images

- `/ov-distros:nvidia` (base for all GPU images)
- `/ov-immich:immich-ml`

## Related Layers

- `/ov-distros:nvidia` — NVIDIA GPU runtime (driver libs, CDI toolkit) — required dependency
- `/ov-selkies:ffmpeg` — FFmpeg multimedia (nonfree codecs) — required dependency
- `/ov-distros:rocm` — AMD GPU counterpart (ROCm runtime + OpenCL)
- `/ov-languages:python-ml` — ML Python environment (depends on cuda)
- `/ov-jupyter:jupyter` — Jupyter notebooks (depends on cuda)
- `/ov-ollama:ollama` — LLM server (depends on cuda)
- `/ov-comfyui:comfyui` — image generation (depends on cuda)

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
