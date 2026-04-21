---
name: fedora-ov
description: |
  Fedora image with the full ov toolchain using shared layers. Rootless-first
  since 2026-04 — runs as uid=1000 with passwordless sudo (no root, no
  cap_add: ALL). Same layer list as arch-ov. Includes NVIDIA GPU runtime.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  the fedora-ov image.
---

# fedora-ov

Fedora container with the full ov toolchain. Uses almost the same
layer list as `/ov-images:arch-ov` — the tag system handles
Fedora-specific packages and scripts via `rpm:` sections. Supports
nested containers at any depth via `/ov-layers:container-nesting`.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora (quay.io/fedora/fedora:43) |
| Tags | `[all, rpm, fedora, fedora:43]` |
| Layers | agent-forwarding, ov-full, golang, gh, sshd, container-nesting, nvidia |
| Platforms | linux/amd64 |
| UID / user | **1000 / user** (rootless-first since 2026-04) |
| Network | host |
| Security | layer-level only (from `/ov-layers:container-nesting`) |
| Registry | ghcr.io/overthinkos |

### Rootless-first posture (2026-04 refactor)

Previously ran as `uid: 0 / user: root` with `cap_add: [ALL]` +
`security_opt: [label=disable, seccomp=unconfined]`. All four
power-user images (`fedora-ov`, `/ov-images:arch-ov`,
`/ov-images:fedora-coder`, `/ov-images:githubrunner`) dropped that
posture once the `/ov-layers:container-nesting` kernel RCA proved
that `unmask=/proc/*` + subuid/subgid delegation is sufficient for
rootless nested containers + rootless libvirt VMs.

The `/ov-layers:sshd` layer installs `/etc/sudoers.d/ov-user` with
passwordless sudo for `user`, so anything that genuinely needs root
is one `sudo` prefix away. Default user for every process is uid=1000.

Resolved OCI security label:

| Field | Value |
|---|---|
| `cap_add` | **(empty)** |
| `security_opt` | `[unmask=/proc/*]` (from `/ov-layers:container-nesting`) |
| `devices` | `[/dev/fuse, /dev/net/tun]` (from `/ov-layers:container-nesting`) |
| `privileged` | `false` |

See `/ov-images:selkies-desktop-ov` (streaming-desktop sibling) and
`/ov-images:fedora-coder` (kitchen-sink dev sibling) for other images
sharing this posture, and `/ov-layers:container-nesting` for the
kernel `mount_too_revealing()` RCA.

### Why still `network: host`

Host networking is kept on `fedora-ov` (unlike `arch-ov` which moved
to bridge) so the image can reach host services and the host
namespace directly. As of 2026-04, ov-mcp's `rewriteMCPURLForHost`
also handles host-networked containers via
`HostConfig.NetworkMode=host` detection (see
`ov/mcp_client.go:lookupHostPort`), so host networking no longer
breaks MCP URL rewriting. If you want ov-mcp on fedora-ov, compose it
into the layer list — it will work on either networking mode.

## What's Installed

Full ov toolchain via shared layers:

- **ov-full** — ov binary + VM tools (qemu-kvm, qemu-img, libvirt tools) + gocryptfs + socat + gvisor-tap-vsock + podman-machine
- **golang** — Go compiler (`golang-bin`)
- **gh** — GitHub CLI + `git` + `git-lfs` (single-responsibility since 2026-04; see `/ov-layers:gh`)
- **sshd** — SSH server/client (`openssh-server`, `openssh-clients`) + passwordless sudo for `user`
- **container-nesting** — buildah, fuse-overlayfs, shadow-utils, skopeo, tailscale, libsecret + nested container config (Tailscale from `tailscale-stable` repo)
- **nvidia** — nvidia-container-toolkit, libva-nvidia-driver (from negativo17 repo; driver libs provided by CDI at runtime)

## Lifecycle

```bash
# Build
ov image build fedora-ov

# Interactive shell (as uid=1000)
ov shell fedora-ov

# Run a command
ov shell fedora-ov -c "ov version"
ov shell fedora-ov -c "sudo -n whoami"    # root (passwordless)

# Start as service
ov start fedora-ov
ov status fedora-ov
ov stop fedora-ov
```

## Nested Containers

Rootless podman works inside fedora-ov at any nesting depth. The
`/ov-layers:container-nesting` layer provides the config + env vars +
subuid/subgid delegation per the podman/stable recipe:

```bash
# Level 1: run containers inside fedora-ov
ov shell fedora-ov -c "podman run --rm quay.io/libpod/alpine:latest echo hello"

# Level 2: run ov inside fedora-ov inside fedora-ov
ov shell fedora-ov -c "ov shell fedora-ov -c 'ov version'"
```

Use `quay.io/libpod/alpine:latest` instead of
`docker.io/library/alpine` to dodge Docker Hub rate limits.

## GPU Support

The `/ov-layers:nvidia` layer provides NVIDIA GPU runtime:

- `nvidia-container-toolkit` — CDI spec generation (driver userspace libs provided by CDI at runtime, matching host kernel module)
- `libva-nvidia-driver` — VA-API acceleration

`ov` automatically calls `EnsureCDI()` before launching GPU
containers. GPU access works at any nesting depth.

## Verification

```bash
ov shell fedora-ov -c "id"                  # uid=1000(user)
ov shell fedora-ov -c "sudo -n whoami"      # root
ov shell fedora-ov -c "ov version"
ov shell fedora-ov -c "ov doctor"
ov shell fedora-ov -c "podman info"
ov shell fedora-ov -c "podman run --rm quay.io/libpod/alpine:latest echo OK"
ov shell fedora-ov -c "which nvidia-ctk"

# Verify OCI tags
ov image inspect fedora-ov --format tags
# ["all","rpm","fedora","fedora:43"]
```

## Unified with arch-ov

Both `fedora-ov` and `/ov-images:arch-ov` use the exact same layer
list. The tag system (`build: [rpm]` + `distro: ["fedora:43", fedora]`
vs `build: [pac]` + `distro: [archlinux]`) selects the right packages
and scripts per distro.

## Key Layers

- `/ov-layers:ov-full` — ov binary plus VM/encryption tools
- `/ov-layers:container-nesting` — nested rootless podman/buildah (authoritative RCA for `mount_too_revealing()` + `unmask=/proc/*`)
- `/ov-layers:sshd` — SSH daemon + passwordless sudo for `user`
- `/ov-layers:gh` — GitHub CLI + git + git-lfs (owns all git tooling as of 2026-04)
- `/ov-layers:nvidia` — NVIDIA GPU runtime

## Related Images

- `/ov-images:fedora` — parent base image
- `/ov-images:arch-ov` — Arch counterpart, same layers, same rootless posture
- `/ov-images:fedora-coder` — kitchen-sink dev sibling (32 layers, adds coding CLIs + DevOps)
- `/ov-images:selkies-desktop-ov` — streaming-desktop counterpart (ov toolchain + browser-accessible Wayland); shares the rootless-first posture
- `/ov-images:githubrunner` — self-hosted GitHub Actions runner; same uid=1000 posture

## Related Commands

- `/ov:shell` — open an interactive shell in fedora-ov (as uid=1000 with sudo)
- `/ov:service` — manage fedora-ov as a service
- `/ov:vm` — nested libvirt VMs via `qemu:///session` (rootless)
- `/ov:mcp` — MCP gateway deployment patterns (if you add `ov-mcp` to the layers)

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
