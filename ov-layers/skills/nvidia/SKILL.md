---
name: nvidia
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
| Install files | `layer.yml`, `tasks:` |
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

The `nvidia-container-toolkit` provides `nvidia-ctk` which generates CDI (Container Device Interface) specs. `ov` calls `EnsureCDI()` before launching containers with GPU — if CDI specs don't exist at `/etc/cdi/nvidia.yaml`, it runs `nvidia-ctk cdi generate` to create them. This enables GPU access in nested containers where host CDI specs are not inherited.

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

`ov` does **not** bake a hardcoded `DRINODE=/dev/dri/renderD128` into this layer. Instead, it auto-detects the correct render node at container-launch time and injects it as an environment variable. The detection + injection is consolidated in a single function, `appendAutoDetectedEnv()` in `ov/devices.go`, which is called by `ov config`, `ov start`, and `ov shell` — so the three commands always produce the same env set.

Selkies is the primary consumer: pixelflux's Wayland compositor uses `DRINODE` to open the render node and set up the VAAPI H.264 encoder. Without the injection, selkies would fall back to software encode (`libx264`) and lose ~40% of its streaming bandwidth budget.

The injection replaces 10 previously-scattered GPU device injection blocks across the `ov` source tree — see commit `8f6f322` for the consolidation history. If you see `DRINODE` referenced in layer scripts, you can assume it was auto-detected and injected by `ov`, not set by the user.

See `/ov:doctor` (Hardware Detection) for the detection probe and `/ov-layers:rocm` for the AMD-side counterpart using the same mechanism.

### Cross-GPU portability (nvidia-base images on AMD hosts)

Images that declare `base: nvidia` (e.g., `/ov-images:selkies-desktop-nvidia`, `/ov-images:selkies-desktop-ov`) still run cleanly on hosts with a different GPU vendor — the NVIDIA runtime libraries ride along as benign passengers. `ov config` auto-detects whatever the host actually exposes (e.g., `/dev/dri/renderD128` + `/dev/kfd` for an AMD RDNA3), injects those device nodes + `DRINODE`, and Mesa handles rendering. Confirmed 2026-04-19: `selkies-desktop-ov` (base: nvidia) on an AMD `gfx 11.0.0` host — 15/15 supervisord programs RUNNING, selkies streaming over Mesa, no CUDA calls attempted. The CUDA toolkit in the image simply goes unused.

## Install tasks

Creates Vulkan ICD compatibility symlinks for nvidia-ctk CDI device injection.

## Used In Images

- `/ov-images:nvidia` — NVIDIA GPU base image (nvidia + cuda layers)
- `/ov-images:arch-ov` — Arch Linux ov toolchain (shared layers + nvidia)
- `/ov-images:fedora-ov` — Fedora ov toolchain (shared layers + nvidia)

## Related Layers

- `/ov-layers:cuda` — CUDA development toolkit (depends on nvidia)
- `/ov-layers:rocm` — AMD GPU counterpart (ROCm runtime + OpenCL), uses the same `appendAutoDetectedEnv()` DRINODE injection
- `/ov-layers:selkies` — Primary consumer of the DRINODE env for VAAPI H.264 encode
- `/ov-layers:python-ml`, `/ov-layers:llama-cpp`, `/ov-layers:jupyter-ml` — CUDA ML stacks that depend on this layer

## Related Commands

- `/ov:doctor` — Host NVIDIA detection (GPU probe, CDI spec status, driver version)
- `/ov:shell` — DRINODE auto-injection applies to interactive shells too
- `/ov:udev` — Device permission management for `/dev/dri/*` and `/dev/nvidia*`
- `/ov:config` — Runtime GPU device injection at deployment time (same `appendAutoDetectedEnv()` path)
- `/ov:start` — Runtime GPU device injection at service start time

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
