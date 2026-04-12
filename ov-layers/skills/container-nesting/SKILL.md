---
name: container-nesting
description: |
  Container-in-container support (podman, buildah, fuse-overlayfs, nested config).
  Use when working with nested containers, container-in-container, or the container-nesting layer.
---

# container-nesting -- Container-in-container support

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages), `root.yml` (nested config) |
| Security | `cap_add: [ALL]`, `label=disable`, `seccomp=unconfined` |
| Volumes | `storage` at `/var/lib/containers/storage` |
| Environment | `OV_BUILD_ENGINE=podman`, `OV_RUN_ENGINE=podman` |

## Packages

**RPM:** `buildah`, `fuse-overlayfs`, `shadow-utils`, `skopeo`, `tailscale`, `libsecret`
(Tailscale from `tailscale-stable` repo)

**Pacman:** `buildah`, `docker`, `fuse-overlayfs`, `shadow`, `skopeo`, `tailscale`, `libsecret`

## Nested Container Configuration (root.yml)

The `all:` task in root.yml configures the container for nested podman/buildah:

1. **subuid/subgid** -- Maps `root:100000:65536` for rootless user namespace support
2. **File capabilities** -- `cap_setuid` on `newuidmap`, `cap_setgid` on `newgidmap`
3. **containers.conf** -- Critical nested container settings:
   - `cgroup_manager = "cgroupfs"` -- systemd cgroup manager unavailable in containers
   - `cgroups = "disabled"` -- cgroup filesystem is read-only at nesting depth 2+
   - `netns = "host"` -- netavark can't create network namespaces inside containers
   - `userns = "host"` -- newuidmap fails in nested user namespaces

## Image-Level Requirements

Images using this layer should also set:
```yaml
uid: 0
gid: 0
user: root
network: host
```

`network: host` at the image level controls the OUTER container's network mode (bypasses netavark sysctl limitations). The `netns=host` in containers.conf controls INNER podman instances.

## Usage

```yaml
# images.yml
my-ov-image:
  base: fedora
  uid: 0
  gid: 0
  user: root
  network: host
  layers:
    - ov-full
    - container-nesting
    - nvidia
```

## Used In Images

- `/ov-images:arch-ov`
- `/ov-images:fedora-ov`

## Related Layers
- `/ov-layers:ov-full` — Pairs with this layer in ov-toolchain images
- `/ov-layers:sshd` — Sibling enabling remote access to nested-container hosts

## Related Commands
- `/ov:build` — Build images that ship podman-in-podman
- `/ov:shell` — Run nested podman/buildah commands inside the outer container
