---
name: arch-ov
description: |
  Arch Linux image with the full ov toolchain. Rootless-first since 2026-04
  — runs as uid=1000 with passwordless sudo (no root, no cap_add: ALL).
  Composes /ov-coder:ov-mcp so the image is reachable as an MCP gateway
  on port 18765. NVIDIA GPU runtime composed in.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  the arch-ov image.
---

# arch-ov

Arch Linux container with the full ov toolchain. Uses the same shared
layer list as `/ov-foundation:fedora-ov` — the tag system handles
Arch-specific packages and scripts via `pac:` sections. Composes
`ov-mcp` so the image is addressable as an MCP gateway — LLM agents
can drive build/test/deploy via Streamable HTTP on port 18765.

## Image Properties

| Property | Value |
|----------|-------|
| Base | archlinux (docker.io/library/archlinux:latest) |
| Tags | `[all, pac, archlinux]` |
| Layers | agent-forwarding, ov-full, **ov-mcp**, golang, gh, sshd, container-nesting, nvidia |
| Platforms | linux/amd64 |
| UID / user | **1000 / user** (rootless-first since 2026-04) |
| Network | default `ov` bridge |
| Ports | `2222:2222` (sshd), `18765:18765` (ov-mcp) |
| Security | layer-level only (from `/ov-foundation:container-nesting`) |
| Registry | ghcr.io/overthinkos |

### Rootless-first posture (2026-04 refactor)

Previously ran as `uid: 0 / user: root` with `cap_add: [ALL]` +
`security_opt: [label=disable, seccomp=unconfined]`. All four
power-user images (`arch-ov`, `/ov-foundation:fedora-ov`,
`/ov-coder:fedora-coder`, `/ov-foundation:githubrunner`) dropped that
posture once the `/ov-foundation:container-nesting` kernel-level RCA
proved that `unmask=/proc/*` + uid-delegation via subuid/subgid ranges
is sufficient for rootless nested containers + rootless libvirt VMs.

The `/ov-coder:sshd` layer installs `/etc/sudoers.d/ov-user` with
passwordless sudo for `user`, so anything that truly needs root inside
the container is one `sudo` prefix away — but the default user for
every process (sshd session, `ov` commands, nested `podman run`) is
uid=1000.

Resolved OCI security label:

| Field | Value |
|---|---|
| `cap_add` | **(empty)** |
| `security_opt` | `[unmask=/proc/*]` (from `/ov-foundation:container-nesting`) |
| `devices` | `[/dev/fuse, /dev/net/tun]` (from `/ov-foundation:container-nesting`) |
| `privileged` | `false` |

See `/ov-selkies:selkies-desktop-ov` for the sibling rootless-first
image that proves this posture works under streaming-desktop +
nested-VM load, and `/ov-foundation:container-nesting` for the kernel
`mount_too_revealing()` RCA.

### Network + port publishing

`network: host` was dropped pre-2026 in favour of the project-default
`ov` bridge — so `ov-mcp`'s MCP URL rewriting
(`rewriteMCPURLForHost` in `ov/mcp_client.go`) has published port
mappings to work with. (As of 2026-04 that function also handles
host-networked containers via `HostConfig.NetworkMode` detection, so
this isn't strictly necessary anymore — but the bridge-network pattern
remains the portable default.) If host-port 2222 is already taken by
another running image (canonical conflict: `/ov-selkies:selkies-desktop-ov`
or any `selkies-desktop-*` variant), remap at config time:
`ov config arch-ov -p 2223:2222`.

### MCP gateway (ov-mcp)

The `/ov-coder:ov-mcp` layer deploys `ov mcp serve --listen :18765`
inside the container under supervisord, advertising ~192 MCP tools
(the full Kong CLI surface, including the project-scaffolding +
YAML-editing + file-write authoring verbs added in 2026). Three
deployment patterns work — bind-mount your project, pin an
`OV_PROJECT_REPO`, or rely on the 2026-04 auto-fallback to
`overthinkos/overthink`:

```bash
# Pattern 1: bind your local checkout
ov config arch-ov --bind project=/home/you/overthink
ov start arch-ov
ov eval mcp call arch-ov image.list.images '{}' --name ov  # lists YOUR project's images

# Pattern 3: no bind-mount (auto-fallback kicks in)
ov config arch-ov
ov start arch-ov
ov eval mcp call arch-ov image.list.images '{}' --name ov  # lists upstream overthinkos/overthink images
```

Volume NAME is `project` (stable bind-mount API); container PATH is
`/workspace`. See `/ov-coder:ov-mcp` for full deployment patterns,
`/ov-build:mcp` Part 2 for the server architecture, and `/ov-core:config`
"Bind-mounting a project checkout for ov mcp serve" for the
bind-mount handshake.

## What's Installed

Full ov toolchain via shared layers:

- **ov-full** — ov binary + VM tools (qemu-full, virtiofsd, libvirt) + gocryptfs + socat
- **ov-mcp** — MCP server exposing the full ov CLI as tools on :18765 (supervisord-managed; bind `project=` for build-mode tools or rely on auto-fallback)
- **golang** — Go compiler (`go`)
- **gh** — GitHub CLI + `git` + `git-lfs` (single-responsibility since 2026-04; see `/ov-coder:gh`)
- **sshd** — SSH server/client (`openssh` on Arch — `package_map` handles the Fedora/Arch name split) + passwordless sudo for `user`
- **container-nesting** — podman, buildah, crun, fuse-overlayfs, skopeo, tailscale, libsecret + nested container config
- **nvidia** — nvidia-utils, nvidia-container-toolkit (CDI generation; benign pacman-hook NVML noise on GPU-less hosts — see `/ov-foundation:nvidia`)

## Lifecycle

```bash
# Build
ov image build arch-ov

# Interactive shell (as uid=1000)
ov shell arch-ov

# Run a command
ov shell arch-ov -c "ov version"
ov shell arch-ov -c "sudo dnf --help"    # sudo works passwordless

# Start as service
ov start arch-ov
ov status arch-ov
ov stop arch-ov
```

## Nested Containers

Rootless podman works inside arch-ov at any nesting depth. The
`/ov-foundation:container-nesting` layer provides the config + env vars +
subuid/subgid delegation (1:999 + 1001:64535 per the podman/stable
recipe):

```bash
# Level 1: run containers inside arch-ov
ov shell arch-ov -c "podman run --rm quay.io/libpod/alpine:latest echo hello"

# Level 2: run ov inside arch-ov inside arch-ov
ov shell arch-ov -c "ov shell arch-ov -c 'ov version'"
```

Use `quay.io/libpod/alpine:latest` instead of `docker.io/library/alpine`
to dodge Docker Hub rate limits — the baked
`container-nesting-alpine-run` test does.

## GPU Support

The `/ov-foundation:nvidia` layer provides NVIDIA GPU runtime:

- `nvidia-utils` — `nvidia-smi` and driver userspace
- `nvidia-container-toolkit` — `nvidia-ctk` for CDI spec generation

`ov` automatically calls `EnsureCDI()` before launching GPU
containers. GPU access works at any nesting depth.

## Verification

```bash
ov shell arch-ov -c "id"                              # uid=1000(user)
ov shell arch-ov -c "sudo -n whoami"                  # root (passwordless)
ov shell arch-ov -c "ov version"
ov shell arch-ov -c "ov doctor"
ov shell arch-ov -c "podman info"
ov shell arch-ov -c "podman run --rm quay.io/libpod/alpine:latest echo OK"
ov shell arch-ov -c "which nvidia-ctk"
```

## Unified with fedora-ov

Both `arch-ov` and `/ov-foundation:fedora-ov` use the exact same layer
list. The tag system (`build: [pac]` + `distro: [archlinux]` vs
`build: [rpm]` + `distro: ["fedora:43", fedora]`) selects the right
packages and scripts per distro.

## Key Layers

- `/ov-coder:ov-full` — ov binary plus VM/encryption tools
- `/ov-coder:ov-mcp` — MCP server gateway (port 18765; `/workspace` bind-mount or auto-fallback)
- `/ov-foundation:container-nesting` — nested podman/buildah (pac: podman + crun + buildah; `docker` is RPM-only — Arch gets podman directly)
- `/ov-coder:sshd` — SSH server/client with `package_map` for cross-distro package names + the passwordless-sudo sudoers drop-in
- `/ov-coder:gh` — GitHub CLI + git + git-lfs (owns all git tooling as of 2026-04)
- `/ov-foundation:nvidia` — NVIDIA GPU runtime

## Related Images

- `/ov-foundation:archlinux` — parent base image
- `/ov-foundation:fedora-ov` — Fedora counterpart, same layers, same rootless posture
- `/ov-coder:fedora-coder` — kitchen-sink dev sibling (32 layers vs 8; adds coding CLIs + DevOps)
- `/ov-selkies:selkies-desktop-ov` — streaming-desktop counterpart (ov toolchain + browser-accessible Wayland); shares the rootless-first posture
- `/ov-foundation:githubrunner` — self-hosted GitHub Actions runner; same uid=1000 posture

## Related Commands

- `/ov-core:shell` — open an interactive shell in arch-ov (as uid=1000 with sudo)
- `/ov-core:service` — manage arch-ov as a service
- `/ov-advanced:vm` — nested libvirt VMs via `qemu:///session` (rootless)
- `/ov-build:eval` — three modes: `ov eval image <ref>` (build-scope, disposable container), `ov eval live <name>` (full-stack against running deployment), `ov eval run <score>` (AI iteration loop)
- `/ov-build:mcp` — MCP gateway + auto-fallback behavior

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
