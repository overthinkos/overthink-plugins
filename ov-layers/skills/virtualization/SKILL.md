---
name: virtualization
description: |
  QEMU/KVM/libvirt virtualization stack with virt-manager and guest tools.
  Use when working with virtual machines, QEMU, KVM, or libvirt.
---

# virtualization -- QEMU/KVM/libvirt stack

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `bindfs`, `genisoimage`, `guestfs-tools`, `libguestfs-tools`, `libvirt-nss`, `passt`, `qemu-guest-agent`, `qemu-img`, `qemu-kvm`, `qemu-system-aarch64`, `qemu-system-arm`, `qemu-system-x86-core`, `tigervnc`, `virt-manager`, `virt-viewer`

## Usage

```yaml
# images.yml
my-vm-host:
  layers:
    - virtualization
```

## Used In Images

- Part of `ov-full` composition layer (used in `githubrunner`)
- `bazzite-ai` (disabled)

## Related Layers

- `/ov-layers:socat` -- part of `ov-full` composition alongside virtualization
- `/ov-layers:gocryptfs` -- part of `ov-full` composition alongside virtualization

## When to Use This Skill

Use when the user asks about:

- QEMU or KVM inside containers
- libvirt or virt-manager
- Virtual machine hosting
- The `virtualization` layer
