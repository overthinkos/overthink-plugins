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
| Install files | `task:`, `candy.yml` |
| Depends | `nvidia`, `ffmpeg` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `CUDA_HOME` | `/usr` |

## Packages

RPM: `cuda-nvcc`, `cuda-cudart-devel`, `cuda-cudart-static`, `cuda-nvrtc-devel`, `cuda-cupti-devel`, `cuda-cccl-devel`, `cuda-cudnn`, `libcurand-devel`, `libcufile-devel`, `onnxruntime`, `libaio-devel`, `cpio`

PAC: `cuda`, `cudnn`, `python-onnxruntime-cpu`

**Note:** FFmpeg codec libraries are provided via the `ffmpeg` dependency layer rather than installed directly. On Fedora, CUDA packages come from the negativo17 `fedora-nvidia` repo (added by the `nvidia` layer); on Arch/CachyOS they come from the standard repos.

## Multi-distro (Fedora + Arch)

The layer is multi-distro. The `distro.arch` section installs `cuda`, `cudnn`,
and `python-onnxruntime-cpu` from the Arch repos. Arch installs CUDA under
`/opt/cuda`, while `CUDA_HOME` is `/usr`, so the Arch path symlinks
`/opt/cuda/*` into `/usr/*` to stitch the Arch layout into the Fedora layout.
With that stitch in place, the same `nvcc`, header, and library paths resolve
identically on Fedora and on Arch/CachyOS.

## Install tasks

Extracts cuDNN headers from `cuda-cudnn-devel` RPM (bypasses driver dependency via rpm2cpio) on Fedora; the `/opt/cuda → /usr` symlink stitch runs on Arch/CachyOS.

## Usage

```yaml
# box.yml
nvidia:
  base: fedora-nonfree
  layers:
    - nvidia
    - cuda
```

## Used In Images

- `/charly-distros:nvidia` (base for all GPU images)
- `/charly-immich:immich-ml`

## Related Layers

- `/charly-distros:nvidia` — NVIDIA GPU runtime (driver libs, CDI toolkit) — required dependency
- `/charly-selkies:ffmpeg` — FFmpeg multimedia (nonfree codecs) — required dependency
- `/charly-distros:rocm` — AMD GPU counterpart (ROCm runtime + OpenCL)
- `/charly-languages:python-ml` — ML Python environment (depends on cuda)
- `/charly-jupyter:jupyter` — Jupyter notebooks (depends on cuda)
- `/charly-ollama:ollama` — LLM server (depends on cuda)
- `/charly-comfyui:comfyui` — image generation (depends on cuda)

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
