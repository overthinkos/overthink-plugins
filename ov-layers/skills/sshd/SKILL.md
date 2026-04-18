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
# image.yml -- typically used via bootc-base composition
my-image:
  layers:
    - sshd
```

## Used In Images

- Part of the `bootc-base` composition layer (used in bootc images)
- `/ov-images:aurora` (disabled)

## Testing Notes

- `/etc/sudoers.d/ov-user` (the NOPASSWD rule written by this layer) is
  `root:root 0750` — the disposable test user (uid 1000) cannot
  traverse `/etc/sudoers.d/`. A `file: /etc/sudoers.d/ov-user; exists: true`
  test reports "missing" even when the file is present. Use
  `command: sudo -n -l; stdout: [{contains: NOPASSWD}]` to verify the
  semantic instead. See `/ov:test` Authoring Gotcha #10.
- Host-side port reachability uses `127.0.0.1:${HOST_PORT:2222}`, not
  `${CONTAINER_IP}:${HOST_PORT:2222}`. See `/ov:test` Gotcha #1.

## Related Skills

- `/ov-layers:bootc-base` -- composition that includes this layer
- `/ov-layers:cloud-init` -- depends on sshd for VM provisioning
- `/ov:test` -- declarative testing framework
- `/ov:layer` -- layer authoring

## When to Use This Skill

Use when the user asks about:

- SSH server setup in containers or VMs
- Port 22 configuration
- OpenSSH server/client packages
- Remote access to bootc images
