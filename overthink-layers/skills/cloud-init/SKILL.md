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
| Install files | `root.yml`, `layer.yml` |

## Packages

- `cloud-init` (RPM) -- cloud instance initialization
- `cloud-utils-growpart` (RPM) -- partition growing utility

## Usage

```yaml
# images.yml
my-cloud-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  layers:
    - cloud-init
```

## Used In Images

Used in bootc images for VM/cloud deployments. Depends on `sshd`.

## Related Layers

- `/overthink-layers:sshd` -- SSH server dependency

## When to Use This Skill

Use when the user asks about:

- Cloud-init configuration or NoCloud datasource
- VM instance provisioning
- Cloud instance bootstrapping
- Partition growing with growpart
- System services: cloud-init-local, cloud-init, cloud-config, cloud-final
