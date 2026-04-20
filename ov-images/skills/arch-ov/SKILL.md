---
name: arch-ov
description: |
  Arch Linux image with full ov toolchain using shared layers. Runs as root on the default `ov` bridge network with cap_add ALL for full container management including nested podman. Ships the `ov-mcp` MCP gateway on port 18765. NVIDIA GPU runtime composed in.
  MUST be invoked before building, deploying, configuring, or troubleshooting the arch-ov image.
---

# arch-ov

Arch Linux container with full ov toolchain. Uses the same shared layer list as `fedora-ov` — the tag system handles Arch-specific packages and scripts via `pac:` sections. Supports running arch-ov inside arch-ov (tested to depth 3). Composes `ov-mcp` so the image is addressable as an MCP gateway — LLM agents can drive build/test/deploy via Streamable HTTP on port 18765.

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux (docker.io/library/archlinux:latest) |
| Tags | `[all, pac, archlinux]` |
| Layers | agent-forwarding, ov-full, **ov-mcp**, golang, gh, sshd, container-nesting, nvidia |
| Platforms | linux/amd64 |
| UID/GID | 0/0 (root) |
| User | root |
| Network | default `ov` bridge (was `host` pre-migration) |
| Ports | `2222:2222` (sshd), `18765:18765` (ov-mcp) |
| Security | cap_add: ALL, security_opt: [unmask=/proc/*, label=disable, seccomp=unconfined] (see "Security posture" below) |
| Registry | ghcr.io/overthinkos |

### Network migration (host → bridge) and port publishing

Historical note: `arch-ov` previously used `network: host`. That was dropped in favour of the project-default `ov` bridge network so `ov-mcp`'s MCP URL rewriting (`rewriteMCPURLForHost` in `ov/mcp_client.go`) has a published host port to map against — host-networked containers expose no `NetworkSettings.Ports` entries, which broke `ov test mcp ping arch-ov`. The explicit `ports: ["2222:2222", "18765:18765"]` block in `image.yml` publishes both SSH and the MCP endpoint onto `127.0.0.1`. If host-port 2222 is already taken (canonical conflict: `selkies-desktop-infra`), remap at config time: `ov config arch-ov -p 2223:2222`. `fedora-ov` is still on `network: host` — it should migrate to the same bridge pattern when ready.

### MCP gateway (ov-mcp)

The `ov-mcp` layer deploys `ov mcp serve --listen :18765` inside the container under supervisord, advertising 176 MCP tools (the full Kong CLI surface) over Streamable HTTP. Project-dir-dependent tools (`image.build`, `image.list.images`, `image.inspect`) work when a project is bind-mounted at `/project` via `--bind project=...`:

```bash
ov config arch-ov --bind project=/home/you/overthink
ov start arch-ov
ov test mcp ping arch-ov --name ov                      # → ok
ov test mcp list-tools arch-ov --name ov | wc -l         # → 176
ov test mcp call arch-ov image.list.images '{}' --name ov  # lists the project's images
```

See `/ov-layers:ov-mcp` for the deployment layer, `/ov:mcp` Part 2 for the server architecture, and `/ov:config` "Bind-mounting a project checkout for ov mcp serve" for the bind-mount handshake.

## What's Installed

Full ov toolchain via shared layers:

- **ov-full** — ov binary + VM tools (qemu-full, virtiofsd, libvirt) + gocryptfs + socat
- **ov-mcp** — MCP server exposing the full ov CLI as tools on :18765 (supervisord-managed; bind `project=` for build-mode tools)
- **golang** — Go compiler (`go`)
- **gh** — GitHub CLI (`github-cli`, `git`)
- **sshd** — SSH server/client (`openssh` on Arch — `package_map` handles the Fedora/Arch name split)
- **container-nesting** — podman, buildah, crun, fuse-overlayfs, skopeo, tailscale, libsecret + nested container config
- **nvidia** — nvidia-utils, nvidia-container-toolkit (CDI generation; benign pacman-hook NVML noise on GPU-less hosts — see `/ov-layers:nvidia`)

## Lifecycle

```bash
# Build
ov image build arch-ov

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

## Security posture (image-level override)

As of the 2026-04-19 `container-nesting` refactor, the **layer** ships
only `security_opt:[unmask=/proc/*] + devices:[/dev/fuse, /dev/net/tun]` —
zero cap_add. The `arch-ov` image asserts the historical full-hammer
posture at the **image level** via `image.yml`:

```yaml
arch-ov:
  security:
    cap_add: [ALL]
    security_opt:
      - label=disable
      - seccomp=unconfined
```

`ov/security.go:66-97` unions image-level values onto the layer-level
merged set, so the resolved OCI label is
`cap_add:[ALL] + security_opt:[unmask=/proc/*, label=disable, seccomp=unconfined] + devices:[/dev/fuse, /dev/net/tun]`.
This preserves the historical permissive posture for root-mode ov
images while letting `/ov-layers:container-nesting` ship a minimum-privilege
baseline for non-root images like `/ov-images:selkies-desktop-ov`.

See `/ov-layers:container-nesting` for the kernel `mount_too_revealing()`
RCA that drove the refactor.

## Nested Containers

Podman works inside arch-ov at any nesting depth. The `container-nesting` layer provides the config + env vars; the image-level `cap_add: [ALL]` is belt-and-braces for root-mode workloads that might need capabilities beyond what rootless delegation grants:

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
- `/ov-layers:ov-mcp` — MCP server gateway (port 18765; `/project` bind-mount for build-mode tools)
- `/ov-layers:container-nesting` — nested podman/buildah (pac: podman + crun + buildah; `docker` is RPM-only now — Arch gets podman directly)
- `/ov-layers:sshd` — SSH server/client with `package_map` for cross-distro package names
- `/ov-layers:nvidia` — NVIDIA GPU runtime

## Related Images
- `/ov-images:archlinux` — parent base image
- `/ov-images:fedora-ov` — Fedora counterpart, same layers
- `/ov-images:selkies-desktop-ov` — non-root counterpart (same `ov` toolchain, no `cap_add` + no host networking; streaming desktop wrapper for browser-accessible operation)
- `/ov-images:githubrunner` — sibling root-mode image for self-hosted CI runners

## Related Commands
- `/ov:shell` — open an interactive shell in arch-ov
- `/ov:service` — manage arch-ov as a service
- `/ov:vm` — nested libvirt VMs via `qemu:///session` (now works rootless too — see `/ov-images:selkies-desktop-ov`)
