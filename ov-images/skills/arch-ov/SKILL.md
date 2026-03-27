---
name: arch-ov
description: |
  Arch Linux image with ov installed from PKGBUILD via makepkg. Runs as root with
  host networking and privileged mode for full container management including nested podman.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch-ov image.
---

# arch-ov

Arch Linux container with ov installed from source via the [overthink-arch](https://github.com/overthinkos/overthink-arch) PKGBUILD. A self-hosting proof: ov building an image that contains itself, installed through the distribution's native package manager.

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux (docker.io/library/archlinux:latest) |
| Layers | arch-ov |
| Platforms | linux/amd64 |
| UID/GID | 0/0 (root) |
| User | root |
| Network | host |
| Security | privileged |
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

Podman works inside arch-ov — the image is configured with privileged mode, rootless podman support, and host networking for nested containers:

```bash
ov shell arch-ov -c "podman run --rm docker.io/library/alpine echo hello"
ov shell arch-ov -c "ov status --all"
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

- **AUR packages not installed:** `docker-buildx`, `cloudflared-bin`, `gvisor-tap-vsock` are AUR-only — the image uses makepkg for ov but doesn't set up yay for additional AUR packages
- **libvirt socket:** Not available inside the container (no virtqemud running) — VM commands that need libvirt will fail
- **Secret storage:** Falls back to plaintext config (no gnome-keyring/keepassxc in container)
- **`config get` fix:** The ov binary built from GitHub may lack the `config get` fix for non-engine keys until the fix is pushed upstream
