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

- `qemu-guest-agent` -- QEMU guest agent daemon. The package name is identical on
  Arch/CachyOS (`pac`) and Fedora (`rpm`), so the top-level `package:` covers both
  — this is a cross-distro layer, not RPM-only.

## Full functionality (all RPCs + fsfreeze)

The layer exposes the **complete** guest-agent surface — `guest-exec`,
`guest-file-*`, `guest-fsfreeze-*`, `guest-set-*`. The package default blocks no
RPCs; the layer's `/etc/qemu/qemu-ga.conf` makes that explicit and turns on the
**fsfreeze hook** (the one capability not active by default), so the host can take
application-consistent snapshots. The hook is a standard dispatcher
(`/etc/qemu/fsfreeze-hook`) that runs every executable in
`/etc/qemu/fsfreeze-hook.d/` with `freeze`/`thaw`; drop per-app scripts there.

The layer also enables the `qemu-guest-agent.service` (system scope) and
contributes the virtio-serial channel (below). On a `kind: vm` entity the channel
is usually declared structurally instead — `channels: [{type: unix, name:
org.qemu.guest_agent.0}]` (see `/ov-internals:libvirt-renderer`).

## Usage

```yaml
# image.yml -- typically used via bootc-base composition
my-vm-image:
  bootc: true
  layers:
    - bootc-base
# or applied to a VM guest at deploy time:
#   ov deploy add vm:<name> qemu-guest-agent
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

This is the `/` classification case: `isDeviceElement` flags it as device-scoped, so the renderer injects it inside `<devices>` rather than before `</domain>`. See `/ov-internals:libvirt-renderer` for the injection pipeline and `/ov-vm:vm` for the QEMU-user-net caveat (the agent shows as enabled/inactive under `ov`'s QEMU backend; libvirt backend activates it).

## Related Layers

- `/ov-distros:bootc-base` -- composition that includes this layer
- `/ov-coder:sshd` -- SSH server (also in bootc-base)
- `/ov-distros:bootc-config` -- bootc system config (also in bootc-base)

## When to Use This Skill

Use when the user asks about:

- QEMU guest agent installation
- Host-guest communication in VMs
- Libvirt virtio channel configuration
- The `org.qemu.guest_agent.0` channel

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations, `libvirt.snippets:`)
- `/ov-vm:vm` — VM lifecycle; bootc VM caveats; QEMU-user-net limitation
- `/ov-vm:vms-catalog` — `kind: vm` entity schema that consumes this layer's contribution
- `/ov-internals:libvirt-renderer` — renderer that injects this layer's snippet into `<devices>`
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
