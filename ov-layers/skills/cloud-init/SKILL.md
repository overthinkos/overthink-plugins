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
# image.yml
my-cloud-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  layers:
    - cloud-init
```

## Used In Images

Used in bootc images for VM/cloud deployments. Depends on `sshd`.

## Related Skills

- `/ov-layers:sshd` -- SSH server dependency
- `/ov-layers:bootc-base` — composition that includes this layer
- `/ov-layers:qemu-guest-agent` — host-guest communication (paired with cloud-init in VMs)
- `/ov:vm` — VM lifecycle (create, start, stop) for bootc disk images
- `/ov:layer` — layer authoring reference

## When to Use This Skill

Use when the user asks about:

- Cloud-init configuration or NoCloud datasource
- VM instance provisioning
- Cloud instance bootstrapping
- Partition growing with growpart
- System services: cloud-init-local, cloud-init, cloud-config, cloud-final

## Related

- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
