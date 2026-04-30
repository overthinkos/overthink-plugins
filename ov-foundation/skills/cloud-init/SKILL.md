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
| Install files | `tasks:`, `layer.yml` |

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

This layer installs the **guest-side** cloud-init package — the daemon that reads a NoCloud seed ISO at first boot. The **host-side** companion is the `RenderCloudInit` path in the `ov` binary itself (`/ov-dev:cloud-init-renderer`), which produces that seed ISO from the structured `VmSpec.CloudInit` block on a `kind: vm` entity.

The two sides cooperate across the host/guest boundary:

- `/ov-dev:cloud-init-renderer` emits `user-data` + `meta-data` + `network-config` + seeds ISO via xorriso.
- This guest-side layer installs cloud-init so the guest reads `/dev/sr0` (the seed ISO) at boot.
- The `composeUsers` adopt-merge pattern (renderer-side) deposits the SSH pubkey in `~<base_user>/.ssh/authorized_keys` without `useradd`.

For cloud_image VMs (`source.kind: cloud_image`), cloud-init typically comes pre-installed in the upstream qcow2 — this layer isn't needed; author the VM entity directly in `vms.yml`. For bootc VMs that want cloud-init provisioning, add this layer explicitly. See `/ov-vms:vms` for the authoring guide and `/ov-vms:arch` for a worked example.

## Related Skills

- `/ov-coder:sshd` -- SSH server dependency
- `/ov-foundation:bootc-base` — composition that includes this layer
- `/ov-foundation:qemu-guest-agent` — host-guest communication (paired with cloud-init in VMs)
- `/ov-advanced:vm` — VM lifecycle (create, start, stop, ssh) for kind:vm entities
- `/ov-vms:vms` — kind:vm authoring reference
- `/ov-dev:cloud-init-renderer` — host-side renderer producing the NoCloud seed ISO this layer reads
- `/ov-build:layer` — layer authoring reference

## When to Use This Skill

Use when the user asks about:

- Cloud-init configuration or NoCloud datasource
- VM instance provisioning
- Cloud instance bootstrapping
- Partition growing with growpart
- System services: cloud-init-local, cloud-init, cloud-config, cloud-final

## Related

- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
