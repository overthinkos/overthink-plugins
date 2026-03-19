---
name: sshd
description: |
  OpenSSH server and client on port 22 for remote access.
  Use when working with SSH access, remote login, or sshd configuration in containers/VMs.
---

# sshd -- OpenSSH server

## Layer Properties

| Property | Value |
|----------|-------|
| Ports | 22 |
| Install files | `layer.yml` |

## Packages

- `openssh-server` (RPM) -- SSH daemon
- `openssh-clients` (RPM) -- SSH client tools (ssh, scp, sftp)

## Usage

```yaml
# images.yml -- typically used via bootc-base composition
my-image:
  layers:
    - sshd
```

## Used In Images

- Part of the `bootc-base` composition layer (used in bootc images)
- `/ov-images:aurora` (disabled)

## Related Layers

- `/ov-layers:bootc-base` -- composition that includes this layer
- `/ov-layers:cloud-init` -- depends on sshd for VM provisioning

## When to Use This Skill

Use when the user asks about:

- SSH server setup in containers or VMs
- Port 22 configuration
- OpenSSH server/client packages
- Remote access to bootc images
