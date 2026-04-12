---
name: arch-ov
description: |
  Arch Linux image with full ov toolchain using shared layers. Runs as root with
  host networking and cap_add ALL for full container management including nested podman.
  Same layer list as fedora-ov. Includes NVIDIA GPU runtime.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch-ov image.
---

# arch-ov

Arch Linux container with full ov toolchain. Uses the same shared layer list as `fedora-ov` — the tag system handles Arch-specific packages and scripts via `pac:` sections and `pac:` tasks in root.yml/user.yml. Supports running arch-ov inside arch-ov (tested to depth 3).

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux (docker.io/library/archlinux:latest) |
| Tags | `[all, pac, archlinux]` |
| Layers | agent-forwarding, ov-full, golang, gh, sshd, container-nesting, nvidia |
| Platforms | linux/amd64 |
| UID/GID | 0/0 (root) |
| User | root |
| Network | host |
| Security | cap_add: ALL, security_opt: label=disable + seccomp=unconfined (from container-nesting) |
| Registry | ghcr.io/overthinkos |

## What's Installed

Full ov toolchain via shared layers:

- **ov-full** — ov binary + VM tools (qemu-full, virtiofsd, libvirt) + gocryptfs + socat
- **golang** — Go compiler (`go`)
- **gh** — GitHub CLI (`github-cli`, `git`)
- **sshd** — SSH server/client (`openssh`)
- **container-nesting** — podman, buildah, docker, fuse-overlayfs, skopeo, tailscale, libsecret + nested container config
- **nvidia** — nvidia-utils, nvidia-container-toolkit (CDI generation)

## Lifecycle

```bash
# Build
ov build arch-ov

# Interactive shell
ov shell arch-ov

# Run a command
ov shell arch-ov -c "ov version"
ov shell arch-ov -c "ov doctor"

# Start as service
ov start arch-ov
ov status arch-ov
ov stop arch-ov
```

## Nested Containers

Podman works inside arch-ov at any nesting depth. The `container-nesting` layer provides `cap_add: ALL` (instead of `privileged: true` to avoid sysfs remount failures) plus `containers.conf` disabling cgroups, user namespaces, and network namespaces for inner podman:

```bash
# Level 1: run containers inside arch-ov
ov shell arch-ov -c "podman run --rm docker.io/library/alpine echo hello"

# Level 2: run ov inside arch-ov inside arch-ov
ov shell arch-ov -c "ov shell arch-ov -c 'ov version'"
```

## GPU Support

The `nvidia` layer provides NVIDIA GPU runtime:
- `nvidia-utils` — `nvidia-smi` and driver userspace
- `nvidia-container-toolkit` — `nvidia-ctk` for CDI spec generation

`ov` automatically calls `EnsureCDI()` before launching GPU containers. GPU access works at any nesting depth.

## Verification

```bash
ov shell arch-ov -c "ov version"
ov shell arch-ov -c "ov doctor"
ov shell arch-ov -c "podman info"
ov shell arch-ov -c "podman run --rm docker.io/library/alpine echo OK"
ov shell arch-ov -c "which nvidia-ctk"
```

## Unified with fedora-ov

Both `arch-ov` and `fedora-ov` use the exact same layer list. The tag system (`build: [pac]` + `distro: [archlinux]` vs `build: [rpm]` + `distro: ["fedora:43", fedora]`) selects the right packages and scripts per distro.

## Key Layers
- `/ov-layers:ov-full` — ov binary plus VM/encryption tools
- `/ov-layers:container-nesting` — nested podman/buildah/docker
- `/ov-layers:nvidia` — NVIDIA GPU runtime

## Related Images
- `/ov-images:archlinux` — parent base image
- `/ov-images:fedora-ov` — Fedora counterpart, same layers

## Related Commands
- `/ov:shell` — open an interactive shell in arch-ov
- `/ov:service` — manage arch-ov as a service
