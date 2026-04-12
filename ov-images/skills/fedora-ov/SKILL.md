---
name: fedora-ov
description: |
  Fedora image with full ov toolchain using shared layers. Runs as root with
  host networking and cap_add ALL for full container management including nested podman.
  Same layer list as arch-ov. Includes NVIDIA GPU runtime.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora-ov image.
---

# fedora-ov

Fedora container with full ov toolchain. Uses the same shared layer list as `arch-ov` — the tag system handles Fedora-specific packages and scripts via `rpm:` sections and `rpm:` tasks in root.yml/user.yml. Supports running fedora-ov inside fedora-ov (nested containers).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora (quay.io/fedora/fedora:43) |
| Tags | `[all, rpm, fedora, fedora:43]` |
| Layers | agent-forwarding, ov-full, golang, gh, sshd, container-nesting, nvidia |
| Platforms | linux/amd64 |
| UID/GID | 0/0 (root) |
| User | root |
| Network | host |
| Security | cap_add: ALL, security_opt: label=disable + seccomp=unconfined (from container-nesting) |
| Registry | ghcr.io/overthinkos |

## What's Installed

Full ov toolchain via shared layers:

- **ov-full** — ov binary + VM tools (qemu-kvm, qemu-img, libvirt tools) + gocryptfs + socat + gvisor-tap-vsock + podman-machine
- **golang** — Go compiler (`golang-bin`)
- **gh** — GitHub CLI (`gh`, `git`)
- **sshd** — SSH server/client (`openssh-server`, `openssh-clients`)
- **container-nesting** — buildah, fuse-overlayfs, shadow-utils, skopeo, tailscale, libsecret + nested container config (Tailscale from `tailscale-stable` repo)
- **nvidia** — nvidia-container-toolkit, libva-nvidia-driver (from negativo17 repo; driver libs provided by CDI at runtime)

## Lifecycle

```bash
# Build
ov build fedora-ov

# Interactive shell
ov shell fedora-ov

# Run a command
ov shell fedora-ov -c "ov version"
ov shell fedora-ov -c "ov doctor"

# Start as service
ov start fedora-ov
ov status fedora-ov
ov stop fedora-ov
```

## Nested Containers

Podman works inside fedora-ov at any nesting depth. The `container-nesting` layer provides `cap_add: ALL` plus `containers.conf` disabling cgroups, user namespaces, and network namespaces for inner podman:

```bash
# Level 1: run containers inside fedora-ov
ov shell fedora-ov -c "podman run --rm docker.io/library/alpine echo hello"

# Level 2: run fedora-ov inside fedora-ov
ov shell fedora-ov -c "ov shell fedora-ov -c 'ov version'"
```

## GPU Support

The `nvidia` layer provides NVIDIA GPU runtime:
- `nvidia-container-toolkit` — CDI spec generation (driver userspace libs provided by CDI at runtime, matching host kernel module)
- `libva-nvidia-driver` — VA-API acceleration

`ov` automatically calls `EnsureCDI()` before launching GPU containers. GPU access works at any nesting depth.

## Verification

```bash
ov shell fedora-ov -c "ov version"
ov shell fedora-ov -c "ov doctor"
ov shell fedora-ov -c "podman info"
ov shell fedora-ov -c "podman run --rm docker.io/library/alpine echo OK"
ov shell fedora-ov -c "which nvidia-ctk"

# Verify OCI tags
ov inspect fedora-ov --format tags
# ["all","rpm","fedora","fedora:43"]
```

## Unified with arch-ov

Both `fedora-ov` and `arch-ov` use the exact same layer list. The tag system (`build: [rpm]` + `distro: ["fedora:43", fedora]` vs `build: [pac]` + `distro: [archlinux]`) selects the right packages and scripts per distro.

## Key Layers
- `/ov-layers:ov-full` — ov binary plus VM/encryption tools
- `/ov-layers:container-nesting` — nested podman/buildah/docker
- `/ov-layers:nvidia` — NVIDIA GPU runtime

## Related Images
- `/ov-images:fedora` — parent base image
- `/ov-images:arch-ov` — Arch counterpart, same layers

## Related Commands
- `/ov:shell` — open an interactive shell in fedora-ov
- `/ov:service` — manage fedora-ov as a service
