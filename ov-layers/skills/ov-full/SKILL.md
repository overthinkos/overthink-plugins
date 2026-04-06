---
name: ov-full
description: |
  Full ov toolchain composition with CLI, virtualization, encrypted storage, and console access.
  Use when working with ov inside containers/VMs, VM management, or encrypted volumes.
---

# ov-full -- Full ov toolchain for containers and VMs

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `ov`, `virtualization`, `gocryptfs`, `socat` |
| Install files | none (pure composition) |

## Packages

- `gvisor-tap-vsock` (RPM) -- networking for podman machine
- `podman-machine` (RPM) -- podman machine support

## Usage

```yaml
# images.yml
my-vm-host:
  layers:
    - ov-full
```

## Related Layers

- `/ov-layers:ov` -- ov CLI binary (included)
- `/ov-layers:virtualization` -- QEMU/KVM/libvirt stack (included)
- `/ov-layers:gocryptfs` -- encrypted filesystem for `ov config` encrypted volumes (included)
- `/ov-layers:socat` -- socket relay for console access and port_relay (included)
- `/ov-layers:bootc-base` -- often paired for OS images

## Used In Images

- `/ov-images:arch-ov`
- `/ov-images:fedora-ov`
- `/ov-images:githubrunner`
- `/ov-images:aurora` (disabled)

## When to Use This Skill

Use when the user asks about:

- Running ov inside containers or VMs
- VM management toolchain
- Encrypted storage inside containers
- Full ov toolchain composition
- Podman machine support
