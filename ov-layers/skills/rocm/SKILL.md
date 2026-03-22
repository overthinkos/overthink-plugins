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

`HSA_OVERRIDE_GFX_VERSION` is **not baked into the layer** -- it is auto-detected from the host GPU at runtime via `/sys/class/kfd/kfd/topology/nodes/*/properties` and injected as a container environment variable. Override with `-e HSA_OVERRIDE_GFX_VERSION=X.Y.Z`.

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
# images.yml -- standalone AMD GPU image
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

- `/ov-layers:cuda` -- NVIDIA GPU counterpart (CUDA toolkit)
- `/ov-layers:python-ml` -- ML Python environment (currently depends on cuda)

## When to Use This Skill

Use when the user asks about:

- AMD GPU support in containers
- ROCm, HIP, or OpenCL setup
- `/dev/kfd` device access
- `HSA_OVERRIDE_GFX_VERSION` configuration
- AMD GPU passthrough or compute
- The rocm layer or AMD GPU computing
- Comparing AMD vs NVIDIA GPU support
