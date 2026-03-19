---
name: cuda
description: |
  CUDA toolkit, cuDNN, ONNX Runtime, and NVIDIA GPU support from negativo17 repos.
  Use when working with GPU computing, CUDA, cuDNN, machine learning infrastructure, or NVIDIA drivers.
---

# cuda -- CUDA toolkit and GPU libraries

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `root.yml`, `layer.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CUDA_HOME` | `/usr` |
| `LD_LIBRARY_PATH` | `/usr/lib64` |

## Packages

RPM (from negativo17 fedora-multimedia repo): `cuda-nvcc`, `cuda-cudart-devel`, `cuda-cudart-static`, `cuda-nvrtc-devel`, `cuda-cupti-devel`, `cuda-cccl-devel`, `cuda-cudnn`, `libcurand-devel`, `libcufile-devel`, `onnxruntime`, `libva-nvidia-driver`, `ffmpeg-libs`, `libaio-devel`, `cpio`

## Usage

```yaml
# images.yml
nvidia:
  base: fedora
  layers:
    - cuda
```

## Used In Images

- `/ov-images:nvidia` (base for all GPU images)
- `/ov-images:immich-cuda`

## Related Layers

- `/ov-layers:python-ml` -- ML Python environment (depends on cuda)
- `/ov-layers:jupyter` -- Jupyter notebooks (depends on cuda)
- `/ov-layers:ollama` -- LLM server (depends on cuda)
- `/ov-layers:comfyui` -- image generation (depends on cuda)

## When to Use This Skill

Use when the user asks about:

- CUDA toolkit or cuDNN setup
- NVIDIA GPU support in containers
- `CUDA_HOME` or `LD_LIBRARY_PATH` environment
- ONNX Runtime configuration
- The nvidia base image or GPU computing
- negativo17 fedora-multimedia repository
