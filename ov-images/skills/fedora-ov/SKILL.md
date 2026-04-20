---
name: fedora-ov
description: |
  Fedora image with full ov toolchain using shared layers. Runs as root with
  host networking and cap_add ALL for full container management including nested podman.
  Same layer list as arch-ov. Includes NVIDIA GPU runtime.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora-ov image.
---

# fedora-ov

Fedora container with full ov toolchain. Uses almost the same layer list as `arch-ov` — the tag system handles Fedora-specific packages and scripts via `rpm:` sections. Supports running fedora-ov inside fedora-ov (nested containers).

### MCP gateway migration note

`arch-ov` composes `ov-mcp` (MCP gateway on :18765) and runs on the default `ov` bridge with explicit `ports:` publishing. `fedora-ov` is the direct sibling but currently still uses `network: host` and does not compose `ov-mcp`. To bring it to parity, mirror `arch-ov`'s `image.yml` stanza:

```yaml
fedora-ov:
  base: fedora
  uid: 0
  gid: 0
  user: root
  ports:                                 # drop `network: host`; publish explicitly
    - "2222:2222"
    - "18765:18765"
  security:
    cap_add: [ALL]
    security_opt:
      - label=disable
      - seccomp=unconfined
  layers:
    - agent-forwarding
    - ov-full
    - ov-mcp                             # add
    - golang
    - gh
    - sshd
    - container-nesting
    - nvidia
```

Rationale for the migration: host-networked containers expose no `NetworkSettings.Ports` entries, which breaks `rewriteMCPURLForHost` in `ov/mcp_client.go` — `ov test mcp ping fedora-ov` would fail with "container port 18765/tcp is not published to a host port". Bridge + explicit `ports:` is the portable pattern that `arch-ov` adopted. See `/ov-images:arch-ov` "Network migration (host → bridge) and port publishing" for the full story and `/ov-layers:ov-mcp` for the layer contract.

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
| Security | cap_add: ALL, security_opt: [unmask=/proc/*, label=disable, seccomp=unconfined] (see "Security posture" below) |
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
ov image build fedora-ov

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

## Security posture (image-level override)

As of the 2026-04-19 `container-nesting` refactor, the **layer** ships
only `security_opt:[unmask=/proc/*] + devices:[/dev/fuse, /dev/net/tun]` —
zero cap_add. The `fedora-ov` image then asserts the historical
full-hammer posture at the **image level** via `image.yml`:

```yaml
fedora-ov:
  security:
    cap_add: [ALL]
    security_opt:
      - label=disable
      - seccomp=unconfined
```

`ov/security.go:66-97` unions image-level values onto the layer-level
merged set (`appendUnique`), so the resolved OCI label is
`cap_add:[ALL] + security_opt:[unmask=/proc/*, label=disable, seccomp=unconfined] + devices:[/dev/fuse, /dev/net/tun]`.
This preserves the historical permissive posture for root-mode ov
images while letting `/ov-layers:container-nesting` ship a minimum-privilege
baseline for non-root images like `/ov-images:selkies-desktop-ov`.

See `/ov-layers:container-nesting` for the `mount_too_revealing()`
kernel RCA and the surgical-vs-hammer fix analysis.

## Nested Containers

Podman works inside fedora-ov at any nesting depth. The `container-nesting` layer provides the config + env vars; the image-level `cap_add: [ALL]` is belt-and-braces for root-mode workloads that might need capabilities beyond what rootless delegation grants.

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
ov image inspect fedora-ov --format tags
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
- `/ov-images:selkies-desktop-ov` — non-root counterpart (same `ov` toolchain, no `cap_add` + no host networking; streaming desktop wrapper for browser-accessible operation)
- `/ov-images:githubrunner` — sibling root-mode image for self-hosted CI runners

## Related Commands
- `/ov:shell` — open an interactive shell in fedora-ov
- `/ov:service` — manage fedora-ov as a service
- `/ov:vm` — nested libvirt VMs via `qemu:///session` (now works rootless too — see `/ov-images:selkies-desktop-ov`)
