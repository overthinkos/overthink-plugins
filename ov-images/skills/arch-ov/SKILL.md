---
name: arch-ov
description: |
  Arch Linux image with ov installed from PKGBUILD via makepkg. Runs as root with
  host networking and cap_add ALL for full container management including nested podman.
  Includes NVIDIA GPU runtime (nvidia layer) for CDI support.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch-ov image.
---

# arch-ov

Arch Linux container with ov installed from source via the [overthink-arch](https://github.com/overthinkos/overthink-arch) PKGBUILD. A self-hosting proof: ov building an image that contains itself, installed through the distribution's native package manager. Supports running arch-ov inside arch-ov (tested to depth 3).

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux (docker.io/library/archlinux:latest) |
| Layers | arch-ov, nvidia |
| Platforms | linux/amd64 |
| UID/GID | 0/0 (root) |
| User | root |
| Network | host |
| Security | cap_add: ALL, security_opt: label=disable + seccomp=unconfined |
| Registry | ghcr.io/overthinkos |

## What's Installed

Full ov toolchain on Arch Linux:

- **ov** — Built from source via makepkg (CalVer version from git)
- **Container engines** — podman, docker, buildah
- **GPU runtime** — nvidia-utils, nvidia-container-toolkit (CDI generation)
- **VM tools** — qemu-full, qemu-img, virtiofsd, libvirt
- **Encrypted storage** — gocryptfs, fuse3
- **Merge/registry** — skopeo
- **Tunnels** — tailscale
- **Build tools** — go, git, base-devel

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

Podman works inside arch-ov at any nesting depth. The image uses `cap_add: ALL` instead of `privileged: true` to avoid sysfs remount failures, plus `containers.conf` disables cgroups, user namespaces, and network namespaces for the inner podman:

```bash
# Level 1: run containers inside arch-ov
ov shell arch-ov -c "podman run --rm docker.io/library/alpine echo hello"
ov shell arch-ov -c "ov status --all"

# Self-hosting: build arch-ov inside arch-ov
ov shell arch-ov -c "cd /tmp && git clone https://github.com/overthinkos/overthink.git && cd overthink/ov && go build -o ../bin/ov . && cd .. && bin/ov build arch-ov"

# Level 2: run ov inside arch-ov inside arch-ov (no --no-auto-detect needed)
ov shell arch-ov -c "ov shell arch-ov -c 'ov version'"
```

## GPU Support

The `nvidia` layer provides NVIDIA GPU runtime:
- `nvidia-utils` — `nvidia-smi` and driver userspace
- `nvidia-container-toolkit` — `nvidia-ctk` for CDI spec generation

`ov` automatically calls `EnsureCDI()` before launching GPU containers — if CDI specs don't exist, it generates them via `nvidia-ctk cdi generate`. This makes GPU access work at any nesting depth.

## Verification

```bash
ov shell arch-ov -c "ov version"          # CalVer output
ov shell arch-ov -c "ov doctor"           # 18+ deps found, exit 0
ov shell arch-ov -c "ov config list"      # All 17 keys
ov shell arch-ov -c "podman info"         # Podman functional
ov shell arch-ov -c "podman run --rm docker.io/library/alpine echo OK"  # Nested containers
ov shell arch-ov -c "which nvidia-ctk"    # CDI toolkit available
```

## Known Limitations

- **AUR packages not installed:** `docker-buildx`, `cloudflared-bin`, `gvisor-tap-vsock` are AUR-only
- **libvirt socket:** Not available inside the container (no virtqemud running)
- **Secret storage:** Falls back to plaintext config (no gnome-keyring/keepassxc in container)
- **NVIDIA driver version:** Container nvidia-utils version must match host driver for `nvidia-smi` to work
