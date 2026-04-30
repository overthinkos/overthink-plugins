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
# image.yml
my-vm-host:
  layers:
    - ov-full
```

## Related Layers

- `/ov-foundation:ov` -- ov CLI binary (included)
- `/ov-foundation:virtualization` -- QEMU/KVM/libvirt stack (included)
- `/ov-foundation:gocryptfs` -- encrypted filesystem for `ov config` encrypted volumes (included)
- `/ov-foundation:socat` -- socket relay for console access and port_relay (included)
- `/ov-foundation:bootc-base` -- often paired for OS images

## Used In Images

- `/ov-coder:arch-ov`
- `/ov-foundation:fedora-ov`
- `/ov-foundation:githubrunner`
- `/ov-foundation:aurora` (disabled)

## When to Use This Skill

Use when the user asks about:

- Running ov inside containers or VMs
- VM management toolchain
- Encrypted storage inside containers
- Full ov toolchain composition
- Podman machine support

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
