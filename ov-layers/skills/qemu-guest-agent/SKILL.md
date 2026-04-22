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
# image.yml -- typically used via bootc-base composition
my-vm-image:
  bootc: true
  layers:
    - bootc-base
```

## Used In Images

Part of the `bootc-base` composition layer. Used transitively in bootc/VM images.

## virtio-serial channel (libvirt XML contribution)

The layer contributes a raw libvirt XML snippet that the libvirt renderer places in the VM's `<devices>` section:

```xml
<channel type='unix'>
  <target type='virtio' name='org.qemu.guest_agent.0'/>
</channel>
```

Emitted in `layer.yml` as:

```yaml
libvirt:
  snippets:
    - "<channel type='unix'><target type='virtio' name='org.qemu.guest_agent.0'/></channel>"
```

This is the `/` classification case: `isDeviceElement` flags it as device-scoped, so the renderer injects it inside `<devices>` rather than before `</domain>`. See `/ov-dev:libvirt-renderer` for the injection pipeline and `/ov:vm` for the QEMU-user-net caveat (the agent shows as enabled/inactive under `ov`'s QEMU backend; libvirt backend activates it).

## Related Layers

- `/ov-layers:bootc-base` -- composition that includes this layer
- `/ov-layers:sshd` -- SSH server (also in bootc-base)
- `/ov-layers:bootc-config` -- bootc system config (also in bootc-base)

## When to Use This Skill

Use when the user asks about:

- QEMU guest agent installation
- Host-guest communication in VMs
- Libvirt virtio channel configuration
- The `org.qemu.guest_agent.0` channel

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations, `libvirt.snippets:`)
- `/ov:vm` — VM lifecycle; bootc VM caveats; QEMU-user-net limitation
- `/ov-vms:vms` — `kind: vm` entity schema that consumes this layer's contribution
- `/ov-dev:libvirt-renderer` — renderer that injects this layer's snippet into `<devices>`
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
