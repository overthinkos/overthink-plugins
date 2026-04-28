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

- `openssh-server` (RPM / deb) — SSH daemon
- `openssh-clients` (RPM) / `openssh-client` (deb, singular) — SSH client tools (ssh, scp, sftp)
- `openssh` (pac) — Arch metapackage bundling both daemon and client
- `sudo` (rpm / pac / deb) — required for the NOPASSWD rule this layer writes

### Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch), `deb:` (Debian/Ubuntu) — full parity across all four supported package-format families. The `openssh-server` / `openssh-clients` naming differs per distro; package-existence tests use `package_map:` to resolve (see below).

**Cross-distro package-test pattern:** the sshd layer's
`openssh-server-package` check uses `package_map:` to resolve the right
name per distro — `openssh-server` on Fedora/Debian, `openssh` on Arch.
This is the canonical worked example for the `package_map` feature; see
`/ov:test` "Cross-distro package names (`package_map:`)" for the
mechanics and the priority ordering (`fedora:43` > `fedora` when both
match).

```yaml
- id: openssh-server-package
  package: openssh-server        # default
  package_map:
    archlinux: openssh
    fedora: openssh-server
    fedora:43: openssh-server
    debian: openssh-server
    ubuntu: openssh-server
  installed: true
```

### Cross-distro sudoers via `getent passwd 1000`

The sudoers drop-in at `/etc/sudoers.d/ov-user` targets the **actual uid-1000 account**, whatever it happens to be named on the running base image. The layer no longer hardcodes a literal `user` — instead it discovers the account name at build time via `getent passwd 1000`:

```yaml
tasks:
  - cmd: |
      account=$(getent passwd 1000 | cut -d: -f1)
      if [ -z "$account" ]; then
        echo "sshd layer: no uid-1000 account found — refusing to write sudoers" >&2
        exit 1
      fi
      printf '%s ALL=(ALL) NOPASSWD: ALL\n' "$account" > /etc/sudoers.d/ov-user
      chmod 0440 /etc/sudoers.d/ov-user
    user: root
```

This works uniformly across both user-policy modes:

| Image | Resolved account | Sudoers content |
|---|---|---|
| fedora-coder, arch-coder, debian-coder | `user` (create mode — `/ov:image` "user_policy") | `user ALL=(ALL) NOPASSWD: ALL` |
| ubuntu-coder | `ubuntu` (adopt mode — `/ov-images:ubuntu` `base_user:`) | `ubuntu ALL=(ALL) NOPASSWD: ALL` |

Why not `${USER}` substitution? The generator substitutes `${USER}` in task fields (paths, URLs, etc.) but **not** inside `cmd:` command text — `cmd:` is passed verbatim to bash, and bash at RUN time doesn't have `$USER` exported. `getent` is the robust, fully-generic alternative. See `/ov:layer` "`${VAR}` substitution scope" and `/ov:image` "user_policy" for the full architectural context.

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

### Dual-mode sudo check — the `runuser -u user --` wrapper

`ov eval image` runs with USER=1000 on container images but USER=0 on bootc images (bootc intentionally keeps USER=root because systemd manages user sessions via login). A naïve `sudo -n -l; contains: NOPASSWD` check fails on bootc — running as root prints root's Defaults block, which doesn't contain the literal string `NOPASSWD`. The layer's current test drops to `user` explicitly when running as root:

```yaml
- id: sudoers-ov-user
  command: |
    if [ "$(id -u)" = "0" ]; then
      runuser -u user -- sudo -n -l
    else
      sudo -n -l
    fi
  exit_status: 0
  stdout:
    - contains: "NOPASSWD"
```

**Portability note:** use `runuser -u user -- <cmd>`, **not**
`runuser -l user -s /bin/bash -c '<cmd>'`. On Arch util-linux (2.42+),
the `-l … -c` form swallows the wrapped command's stdout — reproduced
cleanly: `runuser -l user -s /bin/bash -c 'sudo -n -l'` prints nothing
and exits 0, while `runuser -u user -- sudo -n -l` prints the full
NOPASSWD listing. The layer was fixed to `-u … --` after this was
caught during `arch-ov` bring-up. See `/ov:test` Authoring Gotcha #11.

## Related Skills

- `/ov-layers:bootc-base` -- composition that includes this layer
- `/ov-layers:bootc-config` -- the bootc boot wiring (tty1 autologin + systemd-user supervisord) that typically runs alongside this layer
- `/ov-layers:cloud-init` -- depends on sshd for VM provisioning
- `/ov-images:selkies-desktop-bootc` -- canonical bootc worked example that exercises the dual-mode sudo test
- `/ov-images:ubuntu-coder` -- canonical adopt-mode example; sudoers correctly targets `ubuntu` via getent
- `/ov-images:debian-coder` -- canonical create-mode deb-family example; sudoers targets `user`
- `/ov-images:ubuntu` -- declares the `base_user:` block that makes ubuntu-coder run as `ubuntu`
- `/ov:test` -- declarative testing framework (gotchas #10 and #11, `package_map:`, `exclude_distros:`)
- `/ov:image` -- `user_policy:` field (create / adopt / auto) that drives which account this layer's sudoers targets
- `/ov:layer` -- layer authoring (`${VAR}` substitution scope, cmd: vs write:)

## When to Use This Skill

Use when the user asks about:

- SSH server setup in containers or VMs
- Port 22 configuration
- OpenSSH server/client packages
- Remote access to bootc images
