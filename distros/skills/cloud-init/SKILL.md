---
name: cloud-init
description: |
  Cloud-init for instance initialization in cloud/VM environments with NoCloud datasource.
  Use when working with cloud-init, VM provisioning, or cloud instance bootstrapping.
---

# cloud-init -- cloud instance initialization

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `sshd` |
| Install files | `task:`, `layer.yml` |

## Packages

- `cloud-init` (RPM) -- cloud instance initialization
- `cloud-utils-growpart` (RPM) -- partition growing utility

## Usage

```yaml
# overthink.yml or image.yml
image:
  my-cloud-image:
    base: "quay.io/fedora/fedora-bootc:43"
    bootc: true
    layers:
      - cloud-init
```

## Used In Images

Used in bootc images for VM/cloud deployments. Depends on `sshd`.

## Host-side pairing with `kind: vm` entities

This layer installs the **guest-side** cloud-init package — the daemon that reads a NoCloud seed ISO at first boot. The **host-side** companion is the `RenderCloudInit` path in the `ov` binary itself (`/ov-internals:cloud-init-renderer`), which produces that seed ISO from the structured `VmSpec.CloudInit` block on a `kind: vm` entity.

The two sides cooperate across the host/guest boundary:

- `/ov-internals:cloud-init-renderer` emits `user-data` + `meta-data` + `network-config` + seeds ISO via xorriso.
- This guest-side layer installs cloud-init so the guest reads `/dev/sr0` (the seed ISO) at boot.
- The `composeUsers` adopt-merge pattern (renderer-side) deposits the SSH pubkey in `~<base_user>/.ssh/authorized_keys` without `useradd`.

For cloud_image VMs (`source.kind: cloud_image`), cloud-init typically comes pre-installed in the upstream qcow2 — this layer isn't needed; author the VM entity directly in `vms.yml`. For bootc VMs that want cloud-init provisioning, add this layer explicitly. See `/ov-vm:vms-catalog` for the authoring guide and `/ov-vm:arch` for a worked example.

## Related Skills

- `/ov-coder:sshd` -- SSH server dependency
- `/ov-distros:bootc-base` — composition that includes this layer
- `/ov-distros:qemu-guest-agent` — host-guest communication (paired with cloud-init in VMs)
- `/ov-vm:vm` — VM lifecycle (create, start, stop, ssh) for kind:vm entities
- `/ov-vm:vms-catalog` — kind:vm authoring reference
- `/ov-internals:cloud-init-renderer` — host-side renderer producing the NoCloud seed ISO this layer reads
- `/ov-image:layer` — layer authoring reference

## When to Use This Skill

Use when the user asks about:

- Cloud-init configuration or NoCloud datasource
- VM instance provisioning
- Cloud instance bootstrapping
- Partition growing with growpart
- System services: cloud-init-local, cloud-init, cloud-config, cloud-final

## Related

- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
