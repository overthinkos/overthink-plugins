---
name: dev-tools
description: |
  Developer tools including bat, ripgrep, neovim, gh, direnv, fd-find, htop,
  podman-compose, and many more CLI utilities.
  Use when working with developer tooling, CLI utilities, or container dev environments.
---

# dev-tools -- Developer CLI utilities

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `root.yml` |

## Packages

RPM (with `--setopt=tsflags=noscripts`): `android-tools`, `apptainer`, `apptainer-suid`, `arch-install-scripts`, `asciinema`, `bat`, `bcc`, `bpftop`, `bpftrace`, `bridge-utils`, `debootstrap`, `direnv`, `dislocker`, `ecryptfs-utils`, `fastfetch`, `fd-find`, `fdupes`, `fuse-devel`, `fuse-dislocker`, `fuse3-devel`, `gh`, `git-lfs`, `html2text`, `htop`, `jdupes`, `mosh`, `neovim`, `podman-compose`, `podman-machine`, `podman-remote`, `podman-tui`, `qemu-kvm`, `qemu-user-binfmt`, `qemu-user-static`, `rclone`, `restic`, `ripgrep`, `ShellCheck`, `squashfuse`, `strace`, `sysstat`, `thefuck`, `yamllint`, `yq`, `zoxide`

## Usage

```yaml
# images.yml
my-dev:
  layers:
    - dev-tools
```

## Used In Images

- `/ov-images:fedora-remote` (via remote layer ref)
- `bazzite-ai` (disabled, via `devops-tools` which is separate)

## When to Use This Skill

Use when the user asks about:

- Developer tools in containers
- CLI utilities (bat, ripgrep, fd-find, neovim, gh, etc.)
- Podman tooling inside containers
- The `dev-tools` layer or its packages
