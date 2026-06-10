---
name: nvidia-layer
description: |
  NVIDIA GPU runtime support: driver libs, nvidia-container-toolkit (CDI), and VA-API.
  Fedora (negativo17) and Arch Linux (pac). Base layer for all GPU-accelerated images.
  Use when working with NVIDIA GPU support, CDI device injection, or the nvidia layer.
---

# nvidia -- NVIDIA GPU runtime

NVIDIA runtime layer providing `nvidia-container-toolkit` for CDI device injection and VA-API hardware video acceleration. Driver userspace libraries (libcuda, libnvidia-ml, etc.) are NOT bundled — CDI provides host-matching driver libs at runtime, preventing version mismatches between container and host kernel module. Supports both Fedora and Arch Linux.

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `task:` |
| Depends | none |

## Packages

**RPM** (from negativo17 fedora-multimedia repo):
- `nvidia-container-toolkit` — `nvidia-ctk` CLI for CDI spec generation
- `libva-nvidia-driver` — VA-API hardware video acceleration

**PAC** (Arch Linux):
- `nvidia-utils` — NVIDIA GL/Vulkan userspace, `nvidia-smi`
- `nvidia-container-toolkit` — `nvidia-ctk` CLI for CDI spec generation

## Environment Variables

| Variable | Value |
|----------|-------|
| `LD_LIBRARY_PATH` | `/usr/lib64` (ensures CDI-injected host driver libs are found by the dynamic linker) |

## CDI Support

The `nvidia-container-toolkit` provides `nvidia-ctk` which generates CDI (Container Device Interface) specs. `charly` calls `EnsureCDI()` before launching containers with GPU — if CDI specs don't exist at `/etc/cdi/nvidia.yaml`, it runs `nvidia-ctk cdi generate` to create them. This enables GPU access in nested containers where host CDI specs are not inherited.

### Build-time noise on GPU-less hosts (Arch)

Arch's `nvidia-container-toolkit` ships a pacman post-install hook that
invokes `nvidia-ctk cdi generate`. On a host with no NVIDIA driver
loaded (e.g., an AMD-only build host), the hook fails NVML init:

```
ERROR: failed to generate CDI spec: failed to initialize NVML: Driver Not Loaded
error: command failed to execute correctly
```

This is **benign for the build** — pacman still exits 0 (hooks don't
affect the parent transaction's status), the layer finishes installing,
and the resulting image works at runtime on a GPU-bearing host (where
the CDI spec is generated via `EnsureCDI()` at container-launch time,
not build time). You can ignore the error message. RPM installs don't
trigger the hook, so Fedora-based images don't see this noise.

If you ever build inside CI where even-benign hook errors matter,
either build arch-nvidia images on a GPU-bearing runner, or patch the
layer to carry a build-time `NVIDIA_VISIBLE_DEVICES=void` env var so
nvidia-ctk skips CDI gen.

## DRINODE Auto-Injection

NVIDIA VAAPI acceleration requires the container to know which DRM render node to bind the EGL context against. On multi-GPU hosts there may be `/dev/dri/renderD128`, `/dev/dri/renderD129`, … and the correct one depends on which physical card backs the NVIDIA driver.

`charly` does **not** bake a hardcoded `DRINODE=/dev/dri/renderD128` into this layer. Instead, it auto-detects the correct render node at container-launch time and injects it as an environment variable. The detection + injection is consolidated in a single function, `appendAutoDetectedEnv()` in `charly/devices.go`, which is called by `charly config`, `charly start`, and `charly shell` — so the three commands always produce the same env set.

Selkies is the primary consumer: pixelflux's Wayland compositor uses `DRINODE` to open the render node and set up the VAAPI H.264 encoder. Without the injection, selkies would fall back to software encode (`libx264`) and lose ~40% of its streaming bandwidth budget.

GPU device injection is consolidated into the single `appendAutoDetectedEnv()` function rather than scattered across the `charly` source tree. If you see `DRINODE` referenced in layer scripts, you can assume it was auto-detected and injected by `charly`, not set by the user.

See `/charly-core:charly-doctor` (Hardware Detection) for the detection probe and `/charly-distros:rocm` for the AMD-side counterpart using the same mechanism.

### Cross-GPU portability (nvidia-base images on AMD hosts)

Images that declare `base: nvidia` (e.g., `/charly-selkies:selkies-labwc-nvidia`) still run cleanly on hosts with a different GPU vendor — the NVIDIA runtime libraries ride along as benign passengers. `charly config` auto-detects whatever the host actually exposes (e.g., `/dev/dri/renderD128` + `/dev/kfd` for an AMD RDNA3), injects those device nodes + `DRINODE`, and Mesa handles rendering. For example, `selkies-labwc-nvidia` (base: nvidia) runs on an AMD `gfx 11.0.0` host — all supervisord programs RUNNING, selkies streaming over Mesa, no CUDA calls attempted. The CUDA toolkit in the image simply goes unused.

## Install tasks

Creates Vulkan ICD compatibility symlinks for nvidia-ctk CDI device injection.

## Used In Images

- `/charly-distros:nvidia` — Fedora NVIDIA GPU base image (nvidia + cuda layers)
- `/charly-distros:cachyos` — `cachyos.nvidia`, the CachyOS GPU base (cachyos + agent-forwarding + nvidia + cuda); the nvidia/cuda layers being multi-distro (rpm + pac) is what lets this Arch/CachyOS GPU base reuse them unchanged
- `/charly-coder:charly-arch` — Arch Linux charly toolchain (shared layers + nvidia)
- `/charly-distros:charly-fedora` — Fedora charly toolchain (shared layers + nvidia)

## Related Candies

- `/charly-distros:cuda` — CUDA development toolkit (depends on nvidia)
- `/charly-distros:rocm` — AMD GPU counterpart (ROCm runtime + OpenCL), uses the same `appendAutoDetectedEnv()` DRINODE injection
- `/charly-selkies:selkies` — Primary consumer of the DRINODE env for VAAPI H.264 encode
- `/charly-languages:python-ml`, `/charly-jupyter:llama-cpp`, `/charly-jupyter:jupyter-ml` — CUDA ML stacks that depend on this layer

## Related Commands

- `/charly-core:charly-doctor` — Host NVIDIA detection (GPU probe, CDI spec status, driver version)
- `/charly-core:shell` — DRINODE auto-injection applies to interactive shells too
- `/charly-automation:udev` — Device permission management for `/dev/dri/*` and `/dev/nvidia*`
- `/charly-core:charly-config` — Runtime GPU device injection at deployment time (same `appendAutoDetectedEnv()` path)
- `/charly-core:start` — Runtime GPU device injection at service start time

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
