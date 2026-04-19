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
- `/ov-images:selkies-desktop-bootc` (via bootc-base) — worked example exercising the dual-mode sudo test below

## Testing Notes

- `/etc/sudoers.d/ov-user` (the NOPASSWD rule written by this layer) is
  `root:root 0750` — the non-root test user (uid 1000 in containers)
  cannot traverse `/etc/sudoers.d/`. A `file: /etc/sudoers.d/ov-user; exists: true`
  test reports "missing" even when the file is present. Use
  `command: sudo -n -l; stdout: [{contains: NOPASSWD}]` to verify the
  semantic instead. See `/ov:test` Authoring Gotcha #10.
- Host-side port reachability uses `127.0.0.1:${HOST_PORT:2222}`, not
  `${CONTAINER_IP}:${HOST_PORT:2222}`. See `/ov:test` Gotcha #1.

### Dual-mode sudo check — the `runuser -l user` wrapper

`ov image test` runs with USER=1000 on container images but USER=0 on bootc images (bootc intentionally keeps USER=root because systemd manages user sessions via login). A naïve `sudo -n -l; contains: NOPASSWD` check fails on bootc — running as root prints root's Defaults block, which doesn't contain the literal string `NOPASSWD`. The layer's current test drops to `user` explicitly when running as root:

```yaml
- id: sudoers-ov-user
  command: |
    if [ "$(id -u)" = "0" ]; then
      runuser -l user -s /bin/bash -c 'sudo -n -l'
    else
      sudo -n -l
    fi
  exit_status: 0
  stdout:
    - contains: "NOPASSWD"
```

`runuser -l user -s /bin/bash -c '...'` is the root-available bridge — it honours the target user's login shell and env, and requires no password. This is the canonical pattern for any test that needs to probe user-1000 behaviour on a bootc image. See `/ov:test` Authoring Gotcha #11.

## Related Skills

- `/ov-layers:bootc-base` -- composition that includes this layer
- `/ov-layers:bootc-config` -- the bootc boot wiring (tty1 autologin + systemd-user supervisord) that typically runs alongside this layer
- `/ov-layers:cloud-init` -- depends on sshd for VM provisioning
- `/ov-images:selkies-desktop-bootc` -- canonical bootc worked example that exercises the dual-mode sudo test
- `/ov:test` -- declarative testing framework (gotchas #10 and #11)
- `/ov:layer` -- layer authoring

## When to Use This Skill

Use when the user asks about:

- SSH server setup in containers or VMs
- Port 22 configuration
- OpenSSH server/client packages
- Remote access to bootc images
