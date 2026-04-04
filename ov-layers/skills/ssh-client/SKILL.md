---
name: ssh-client
description: |
  OpenSSH client tools for SSH agent forwarding.
  Use when working with SSH client, SSH agent forwarding, or the ssh-client layer.
---

# ssh-client -- OpenSSH client tools

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `openssh-clients`
PAC: `openssh`

Note: On Arch Linux, `openssh` is a single package that includes both client and server. The `sshd` layer also installs this package but adds a systemd service and opens port 22.

## Usage

```yaml
# images.yml
my-image:
  layers:
    - ssh-client
```

Typically used as part of the `agent-forwarding` composition layer rather than directly. Use the `sshd` layer instead if you need an SSH *server*.

## Runtime Behavior

Provides `ssh`, `ssh-add`, `ssh-keygen`, `ssh-agent`, `scp`, `sftp` binaries. When combined with SSH agent forwarding (`ov shell`, `ov start` direct mode), the container's SSH commands use the host's SSH agent via a forwarded socket at `/run/host-ssh-auth.sock`.

No SSH agent runs inside the container — the `SSH_AUTH_SOCK` environment variable points to the forwarded host socket.

## Used In Images

Part of `agent-forwarding` composition layer, used in 27 application images.

## Related Layers

- `/ov-layers:agent-forwarding` -- metalayer that includes gnupg + direnv + ssh-client
- `/ov-layers:sshd` -- SSH server + client (includes systemd service, port 22)
- `/ov-layers:gh` -- GitHub CLI + git (uses SSH for git operations)

## When to Use This Skill

Use when the user asks about:

- SSH client tools inside containers
- SSH agent forwarding
- The `ssh-client` layer
- The difference between `ssh-client` and `sshd` layers
