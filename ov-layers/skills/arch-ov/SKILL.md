---
name: arch-ov
description: |
  Arch Linux layer that installs ov from PKGBUILD via makepkg, with full container engine
  support (podman, docker, buildah), VM tools, encrypted storage, and nested container support.
  Use when working with the arch-ov layer.
---

# arch-ov

Installs ov from the [overthink-arch](https://github.com/overthinkos/overthink-arch) PKGBUILD via `makepkg`, with all ov runtime dependencies for full functionality inside a container — including running containers inside containers.

## Layer Properties

| Property | Value |
|----------|-------|
| Package format | pac |
| Packages | base-devel, go, git, podman, buildah, docker, shadow, fuse-overlayfs, qemu-full, qemu-img, virtiofsd, libvirt, gocryptfs, fuse3, skopeo, tailscale, libsecret, openssh, util-linux |
| Security | `cap_add: [ALL]`, `security_opt: [label=disable, seccomp=unconfined]` |
| Volumes | `storage` at `/var/lib/containers/storage` |
| Environment | `OV_BUILD_ENGINE=podman`, `OV_RUN_ENGINE=podman` |

## How ov Gets Installed

The `root.yml` Taskfile:

1. Creates a temporary `_builder` user (makepkg refuses to run as root)
2. Clones `https://github.com/overthinkos/overthink-arch.git` (contains the PKGBUILD)
3. Runs `makepkg --noconfirm` which:
   - Clones the overthink repo as source
   - Builds ov with `CGO_ENABLED=0 go build -trimpath`
   - Creates a pacman package with CalVer versioning
4. Installs the package via `pacman -U`
5. Configures rootless podman: subuid/subgid for root, setcap on newuidmap/newgidmap
6. Writes `/etc/containers/containers.conf` for nested podman support

## Nested Container Support

The layer configures podman-in-podman at any nesting depth:

- `cap_add: ALL` + `security_opt: [label=disable, seccomp=unconfined]` — grants all capabilities without triggering sysfs remount (which `--privileged` does and which fails at depth 2+)
- `cgroups = "disabled"` in containers.conf — cgroup filesystem is read-only inside nested containers
- `userns = "host"` — newuidmap can't write to uid_map through nested user namespaces
- `netns = "host"` — netavark can't create network namespaces inside containers
- `cgroup_manager = "cgroupfs"` — systemd cgroup manager unavailable in containers
- `/var/lib/containers/storage` volume persists podman's internal data

## Why cap_add ALL instead of privileged

`--privileged` tells crun to mount `/sys`, `/proc`, and other special filesystems fresh. At nesting depth 2+, the kernel blocks this sysfs remount. `--cap-add=ALL` + `--security-opt label=disable` + `--security-opt seccomp=unconfined` grants equivalent permissions without the sysfs mount attempt — making nested containers work at any depth.

## Dependencies

None (standalone layer — all packages installed via pac section).

## Source Files

- `layers/arch-ov/layer.yml` — Package list, security, volumes, env
- `layers/arch-ov/root.yml` — makepkg install + podman configuration
- `pkg/arch/PKGBUILD` — AUR package definition (submodule: overthinkos/overthink-arch)
