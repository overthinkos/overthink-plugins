---
name: arch-ov
description: |
  Arch Linux image with ov installed from PKGBUILD via makepkg. Runs as root with
  host networking and cap_add ALL for full container management including nested podman.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch-ov image.
---

# arch-ov

Arch Linux container with ov installed from source via the [overthink-arch](https://github.com/overthinkos/overthink-arch) PKGBUILD. A self-hosting proof: ov building an image that contains itself, installed through the distribution's native package manager. Supports running arch-ov inside arch-ov (tested to depth 3).

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux (docker.io/library/archlinux:latest) |
| Layers | arch-ov |
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

# Level 2: run ov inside arch-ov inside arch-ov
ov shell arch-ov -c "ov shell arch-ov --no-auto-detect -c 'ov version'"
```

## Verification

```bash
ov shell arch-ov -c "ov version"          # CalVer output
ov shell arch-ov -c "ov doctor"           # 18+ deps found, exit 0
ov shell arch-ov -c "ov config list"      # All 17 keys
ov shell arch-ov -c "podman info"         # Podman functional
ov shell arch-ov -c "podman run --rm docker.io/library/alpine echo OK"  # Nested containers
```

## Known Limitations

- **AUR packages not installed:** `docker-buildx`, `cloudflared-bin`, `gvisor-tap-vsock` are AUR-only
- **libvirt socket:** Not available inside the container (no virtqemud running)
- **Secret storage:** Falls back to plaintext config (no gnome-keyring/keepassxc in container)
- **GPU at level 2:** Use `--no-auto-detect` when running `ov shell` at level 2 to avoid CDI device resolution errors for NVIDIA GPUs
