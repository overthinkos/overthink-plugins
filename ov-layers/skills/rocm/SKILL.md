---
name: rocm
description: |
  AMD ROCm runtime, OpenCL, and GPU compute support via system packages.
  Use when working with AMD GPU computing, ROCm, HIP, OpenCL, or AMD GPU passthrough in containers.
---

# rocm -- AMD ROCm runtime and OpenCL

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `ROCM_PATH` | `/usr` |

## Runtime Environment (auto-detected)

| Variable | Source | Example |
|----------|--------|---------|
| `HSA_OVERRIDE_GFX_VERSION` | KFD topology sysfs | `10.3.0` (RDNA2), `11.0.0` (RDNA3) |
| `DRINODE` | DRM render node enumeration | `/dev/dri/renderD128` (typical), `/dev/dri/renderD129` (multi-GPU) |

Both variables are **not baked into the layer** — they are auto-detected from host state at runtime and injected as container environment variables via `appendAutoDetectedEnv()` in `ov/devices.go`. The same function is called by `ov config`, `ov start`, and `ov shell`, so interactive shells and deployed services see the identical env set.

- `HSA_OVERRIDE_GFX_VERSION` is read from `/sys/class/kfd/kfd/topology/nodes/*/properties` (gfx_target_version field).
- `DRINODE` is selected by walking `/dev/dri/renderD*` and picking the node that matches the AMD PCI device exposed to the container.

Override either with `-e HSA_OVERRIDE_GFX_VERSION=X.Y.Z` or `-e DRINODE=/dev/dri/renderD129`. See `/ov:doctor` (Hardware Detection) for how the probe runs on the host side, and `/ov-layers:nvidia` (DRINODE Auto-Injection) for the NVIDIA counterpart using the same mechanism.

## Security

```yaml
security:
  group_add:
    - keep-groups
```

Uses `keep-groups` to preserve host supplementary groups (video, render) inside the container. This is the standard approach across all layers -- Podman's `keep-groups` is mutually exclusive with explicit group names.

## Packages

RPM (Fedora system repos): `rocm-hip-runtime`, `rocm-opencl`, `rocm-clinfo`, `rocm-smi`

## Host Requirements

AMD GPU support requires:
1. `/dev/kfd` device (auto-detected by `ov`)
2. `/dev/dri/renderD*` render nodes (auto-detected)
3. User in `video` and `render` groups (`ov udev status` to check)
4. `amdgpu` kernel driver loaded

Run `ov doctor` to verify detection. Run `ov udev install` to set up device permissions.

## Usage

```yaml
# image.yml -- standalone AMD GPU image
my-amd-app:
  base: fedora
  layers:
    - rocm
    - my-app
```

## Verification

```bash
# Check AMD GPU detected on host
ov doctor | grep "AMD GPU"

# Verify inside container
ov shell my-amd-app -c "clinfo --list"
ov shell my-amd-app -c "rocm-smi"
ov shell my-amd-app -c "echo \$HSA_OVERRIDE_GFX_VERSION"
```

## Related Layers

- `/ov-layers:nvidia` -- NVIDIA GPU counterpart (runtime libs + CDI), shares `appendAutoDetectedEnv()` DRINODE injection
- `/ov-layers:cuda` -- NVIDIA CUDA toolkit (stacked on nvidia)
- `/ov-layers:python-ml` -- ML Python environment (currently depends on cuda; ROCm equivalent is a future direction)

## Related Commands

- `/ov:doctor` -- Host AMD GPU detection (`/dev/kfd`, render nodes, driver status)
- `/ov:shell` -- Interactive shells receive the same auto-detected HSA_OVERRIDE_GFX_VERSION + DRINODE envs
- `/ov:udev` -- Device permission management for `/dev/kfd` and `/dev/dri/renderD*`
- `/ov:config` -- Runtime GPU env injection at deployment time (same auto-detect path)
- `/ov:start` -- Runtime GPU env injection at service start time

## Used In Images

Not directly used in any current image definition. Available as a standalone layer for AMD GPU support. The NVIDIA base image (`/ov-images:nvidia`) is the currently-shipped GPU image; an AMD counterpart can be composed by substituting this layer.

## When to Use This Skill

Use when the user asks about:

- AMD GPU support in containers
- ROCm, HIP, or OpenCL setup
- `/dev/kfd` device access
- `HSA_OVERRIDE_GFX_VERSION` configuration
- AMD GPU passthrough or compute
- The rocm layer or AMD GPU computing
- Comparing AMD vs NVIDIA GPU support
