---
name: dev-tools
description: |
  Developer tools including bat, ripgrep, neovim, direnv, fd-find, htop,
  podman-compose, and many more CLI utilities.
  Use when working with developer tooling, CLI utilities, or container dev environments.
---

# dev-tools -- Developer CLI utilities

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `candy.yml`, `task:` |

## Packages

RPM (with `--setopt=tsflags=noscripts`): `android-tools`, `apptainer`, `apptainer-suid`, `arch-install-scripts`, `asciinema`, `bat`, `bcc`, `bpftop`, `bpftrace`, `bridge-utils`, `debootstrap`, `direnv`, `dislocker`, `ecryptfs-utils`, `fastfetch`, `fd-find`, `fdupes`, `fuse-devel`, `fuse-dislocker`, `fuse3-devel`, `html2text`, `htop`, `jdupes`, `mosh`, `neovim`, `podman-compose`, `podman-machine`, `podman-remote`, `podman-tui`, `qemu-kvm`, `qemu-user-binfmt`, `qemu-user-static`, `rclone`, `restic`, `ripgrep`, `ShellCheck`, `squashfuse`, `strace`, `sysstat`, `thefuck`, `yamllint`, `yq`, `zoxide`

### Cross-distro coverage

`rpm:` (Fedora), `pac:` (Arch), `deb:` (Debian + generic Ubuntu), with a `ubuntu:24.04:` tag-section override. Not every package is available everywhere; per-distro drops are documented inline in `candy.yml`.

**Packages dropped per distro** (non-exhaustive):

- Arch: `apptainer-suid`, `dislocker`, `ecryptfs-utils`, `fuse-dislocker`, `podman-machine`, `podman-remote`, `podman-tui` — not packaged.
- Debian: same drops as Arch, plus `arch-install-scripts` (obviously) and `apptainer` itself (only in bookworm-backports, skipped on trixie).
- Ubuntu 24.04: additionally drops `fastfetch` — package not in noble main. Handled by a `ubuntu:24.04:` tag section that re-lists the deb packages minus fastfetch.

### bat → batcat symlink (Debian/Ubuntu)

Both Debian and Ubuntu rename `bat` → `batcat` in their archives to avoid a namespace collision with a legacy `bacula` utility. The `bat` package installs only `/usr/bin/batcat`; nothing lives at `/usr/bin/bat`. To keep downstream scripts, docs, and declarative tests portable, the layer ships a distro-tolerant symlink task:

```yaml
task:
  - cmd: |
      if [ -f /usr/bin/batcat ] && [ ! -e /usr/bin/bat ]; then
        ln -sf /usr/bin/batcat /usr/bin/bat
      fi
    user: root
```

No-op on Fedora/Arch (where `/usr/bin/bat` already exists from the distro package) — the guard short-circuits. Creates the symlink on Debian/Ubuntu. Idempotent across rebuilds.

### fastfetch test — `exclude_distros: [ubuntu:24.04]`

The `fastfetch-binary` test is declared with an `exclude_distros:` filter:

```yaml
- id: fastfetch-binary
  file: /usr/bin/fastfetch
  exists: true
  exclude_distros:
    - ubuntu:24.04
```

On images whose `ai.opencharly.platform.distro` OCI label includes `ubuntu:24.04`, the test runner skips this check with a reason — see `/charly-eval:eval` "`exclude_distros:` field". This was added because dropping fastfetch from the `ubuntu:24.04:` tag section is clean, but the baked test probe would otherwise false-fail.

### Git tooling lives in `/charly-coder:gh`

dev-tools does NOT install `gh`, `git`, or `git-lfs` — those belong
exclusively to `/charly-coder:gh`, the dedicated GitHub / git tooling layer.
That keeps ownership unambiguous (one layer installs and tests them). If
you want the git tooling, compose `/charly-coder:gh` alongside
`/charly-coder:dev-tools`.

## Usage

```yaml
# box.yml
my-dev:
  layers:
    - dev-tools
```

## Used In Images

- `bazzite` (via `devops-tools` which is separate)

## Related Layers
- `/charly-coder:gh` — Sibling GitHub CLI commonly paired with dev-tools
- `/charly-coder:devops-tools` — Sibling DevOps cloud CLI bundle in bootc images
- `/charly-coder:build-toolchain` — Sibling C/C++ toolchain in bootc image stacks

## Related Images
- `/charly-coder:debian-coder`, `/charly-coder:ubuntu-coder` — canonical consumers of the bat→batcat symlink
- `/charly-coder:fedora-coder`, `/charly-coder:arch-coder` — consumers where the symlink task is a harmless no-op

## Related Commands
- `/charly-build:build` — Build images that ship the dev-tools package set
- `/charly-core:shell` — Interactive shell to use bat/ripgrep/neovim/etc.
- `/charly-eval:eval` — `exclude_distros:` field reference for the fastfetch-binary test
- `/charly-image:layer` — authoring reference for distro-tolerant `cmd:` tasks and `exclude_distros:`

## When to Use This Skill

Use when the user asks about:

- Developer tools in containers
- CLI utilities (bat, ripgrep, fd-find, neovim, etc.)
- Podman tooling inside containers
- The `dev-tools` layer or its packages
