---
name: qemu-guest-agent
description: |
  QEMU guest agent for host-guest communication in virtual machines.
  Use when working with QEMU/KVM VMs, guest agent setup, or libvirt channel configuration.
---

# qemu-guest-agent -- QEMU guest agent

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` |

## Packages

- `qemu-guest-agent` (RPM) -- QEMU guest agent daemon

## Usage

```yaml
# images.yml -- typically used via bootc-base composition
my-vm-image:
  bootc: true
  layers:
    - bootc-base
```

## Used In Images

Part of the `bootc-base` composition layer. Used transitively in bootc/VM images.

## Related Layers

- `/overthink-layers:bootc-base` -- composition that includes this layer
- `/overthink-layers:sshd` -- SSH server (also in bootc-base)
- `/overthink-layers:bootc-config` -- bootc system config (also in bootc-base)

## When to Use This Skill

Use when the user asks about:

- QEMU guest agent installation
- Host-guest communication in VMs
- Libvirt virtio channel configuration
- The `org.qemu.guest_agent.0` channel
