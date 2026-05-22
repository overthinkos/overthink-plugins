---
name: fedora-ov
description: |
  Fedora image with the full ov toolchain using shared layers. Rootless-first —
  runs as uid=1000 with passwordless sudo (no root, no cap_add: ALL). Same layer
  list as arch-ov. Includes NVIDIA GPU runtime.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  the fedora-ov image.
---

# fedora-ov

Fedora container with the full ov toolchain. Uses almost the same
layer list as `/ov-coder:arch-ov` — the tag system handles
Fedora-specific packages and scripts via `rpm:` sections. Supports
nested containers at any depth via `/ov-distros:container-nesting`.

Lives in the **`overthinkos/fedora`** repo (git submodule at **`image/fedora`**),
`enabled: false`. Its `fedora` base is remote-included from the main repo's
`fedora-base.yml`; its layers (incl. the `nvidia` layer) are pulled by github
reference. Build from the submodule:
`ov -C image/fedora image build fedora-ov --include-disabled`. Deploy-mode verbs
work from anywhere once the image is in local storage.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora (quay.io/fedora/fedora:43) |
| Tags | `[all, rpm, fedora, fedora:43]` |
| Layers | agent-forwarding, ov-full, golang, gh, sshd, container-nesting, nvidia |
| Platforms | linux/amd64 |
| UID / user | **1000 / user** (rootless-first) |
| Network | host |
| Security | layer-level only (from `/ov-distros:container-nesting`) |
| Registry | ghcr.io/overthinkos |

### Rootless-first posture

Runs as uid=1000 / `user` with no added capabilities. The
`/ov-distros:container-nesting` kernel RCA establishes that
`unmask=/proc/*` + subuid/subgid delegation is sufficient for rootless
nested containers + rootless libvirt VMs, so this image — like its
power-user siblings (`/ov-coder:arch-ov`, `/ov-coder:fedora-coder`,
`/ov-distros:githubrunner`) — needs no `cap_add: [ALL]` or
`label=disable` / `seccomp=unconfined`.

The `/ov-coder:sshd` layer installs `/etc/sudoers.d/ov-user` with
passwordless sudo for `user`, so anything that genuinely needs root
is one `sudo` prefix away. Default user for every process is uid=1000.

Resolved OCI security label:

| Field | Value |
|---|---|
| `cap_add` | **(empty)** |
| `security_opt` | `[unmask=/proc/*]` (from `/ov-distros:container-nesting`) |
| `devices` | `[/dev/fuse, /dev/net/tun]` (from `/ov-distros:container-nesting`) |
| `privileged` | `false` |

See `/ov-openclaw:openclaw-desktop` (streaming-desktop sibling) and
`/ov-coder:fedora-coder` (kitchen-sink dev sibling) for other images
sharing this posture, and `/ov-distros:container-nesting` for the
kernel `mount_too_revealing()` RCA.

### Why still `network: host`

`fedora-ov` uses host networking (unlike `arch-ov`, which uses bridge)
so the image can reach host services and the host namespace directly.
ov-mcp's `rewriteMCPURLForHost` handles host-networked containers via
`HostConfig.NetworkMode=host` detection (see
`ov/mcp_client.go:lookupHostPort`), so host networking does not break
MCP URL rewriting. If you want ov-mcp on fedora-ov, compose it
into the layer list — it will work on either networking mode.

## What's Installed

Full ov toolchain via shared layers:

- **ov-full** — ov binary + VM tools (qemu-kvm, qemu-img, libvirt tools) + gocryptfs + socat + gvisor-tap-vsock + podman-machine
- **golang** — Go compiler (`golang-bin`)
- **gh** — GitHub CLI + `git` + `git-lfs` (single-responsibility; see `/ov-coder:gh`)
- **sshd** — SSH server/client (`openssh-server`, `openssh-clients`) + passwordless sudo for `user`
- **container-nesting** — buildah, fuse-overlayfs, shadow-utils, skopeo, tailscale, libsecret + nested container config (Tailscale from `tailscale-stable` repo)
- **nvidia** — nvidia-container-toolkit, libva-nvidia-driver (from negativo17 repo; driver libs provided by CDI at runtime)

## Lifecycle

```bash
# Build (from the overthinkos/fedora submodule; disabled image)
ov -C image/fedora image build fedora-ov --include-disabled

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
`/ov-distros:container-nesting` layer provides the config + env vars +
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

The `/ov-distros:nvidia` layer provides NVIDIA GPU runtime:

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

Both `fedora-ov` and `/ov-coder:arch-ov` use the exact same layer
list. The tag system (`build: [rpm]` + `distro: ["fedora:43", fedora]`
vs `build: [pac]` + `distro: [arch]`) selects the right packages
and scripts per distro.

## Key Layers

- `/ov-coder:ov-full` — ov binary plus VM/encryption tools
- `/ov-distros:container-nesting` — nested rootless podman/buildah (authoritative RCA for `mount_too_revealing()` + `unmask=/proc/*`)
- `/ov-coder:sshd` — SSH daemon + passwordless sudo for `user`
- `/ov-coder:gh` — GitHub CLI + git + git-lfs (owns all git tooling)
- `/ov-distros:nvidia` — NVIDIA GPU runtime

## Related Images

- `/ov-distros:fedora` — parent base image
- `/ov-coder:arch-ov` — Arch counterpart, same layers, same rootless posture
- `/ov-coder:fedora-coder` — kitchen-sink dev sibling (32 layers, adds coding CLIs + DevOps)
- `/ov-openclaw:openclaw-desktop` — streaming-desktop counterpart (ov toolchain + browser-accessible Wayland); shares the rootless-first posture
- `/ov-distros:githubrunner` — self-hosted GitHub Actions runner; same uid=1000 posture

## Related Commands

- `/ov-core:shell` — open an interactive shell in fedora-ov (as uid=1000 with sudo)
- `/ov-core:service` — manage fedora-ov as a service
- `/ov-vm:vm` — nested libvirt VMs via `qemu:///session` (rootless)
- `/ov-build:ov-mcp-cmd` — MCP gateway deployment patterns (if you add `ov-mcp` to the layers)

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
