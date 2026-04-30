---
name: arch-coder
description: |
  Kitchen-sink development image on Arch Linux: coding + AI-coding CLIs +
  DevOps tooling in one container. Arch base, 30+ direct layers mirroring
  fedora-coder's stack but with pac:-section packages (plus AUR for a few
  unique cases). Runs as uid 1000 (`user`) with passwordless sudo.
  Use when working with the arch-coder image — or when comparing
  cross-distro parity across the four coder-family images (fedora,
  debian, ubuntu, arch).
---

# arch-coder

Arch Linux counterpart of `/ov-coder:fedora-coder`. Same daily-development surface (AI CLIs, language runtimes, DevOps stack), same rootless posture (uid 1000 + passwordless sudo), same shape of tests — only the package manager (`pac:` / `aur:`) and a handful of Arch-specific package names differ.

## Definition

```yaml
arch-coder:
  base: archlinux
  ports:
    - "2222:2222"                 # sshd-wrapper
    - "18765:18765"               # ov-mcp (Streamable HTTP)
  layers:
    # Baseline — identical to fedora-coder
    - agent-forwarding
    - sshd
    - ov-full
    - ov-mcp
    - container-nesting
    - dbus
    - tmux

    # Language runtimes + managers
    - language-runtimes           # Go + PHP + .NET 9 + python3-devel
    - golang
    - nodejs                      # NOT nodejs24 — Arch's nodejs is already current
    - rust
    - pixi
    - uv

    # Build + developer tooling
    - build-toolchain
    - dev-tools
    - pre-commit
    - direnv
    - gh
    - typst
    - asciinema

    # AI coding CLIs
    - claude-code
    - codex
    - gemini
    - forgecode
    - oracle

    # DevOps / cloud / infra
    - devops-tools
    - kubernetes
    - docker-ce
    - github-actions
    - google-cloud
    - google-cloud-npm
    - grafana-tools
```

No explicit `user:` / `uid:` / `gid:` — inherits the defaults. `user_policy:` defaults to `auto`; the `archlinux` base image declares no `base_user:` in `build.yml`, so the policy falls through to create mode — the bootstrap emits an idempotent `useradd -m -u 1000 -g 1000 user`. See `/ov-build:image` "user_policy" and `/ov-build:build` "base_user".

## Resolved security posture

| Field | Value | Source |
|---|---|---|
| `cap_add` | **(empty)** | — |
| `security_opt` | `[unmask=/proc/*]` | `/ov-foundation:container-nesting` |
| `devices` | `[/dev/fuse, /dev/net/tun]` | `/ov-foundation:container-nesting` |
| `privileged` | `false` | — |
| Network | bridge | defaults |
| UID / user | `1000 / user` | create mode |
| sudo | passwordless via `/etc/sudoers.d/ov-user` | `/ov-coder:sshd` |

**No `--privileged`, no `cap_add: ALL`, no `seccomp=unconfined`.** The rootless posture is shared with the three sibling power-user images (`fedora-coder`, `fedora-ov`, `arch-ov`) — dropped the legacy privileged posture in 2026-04 once `/ov-foundation:container-nesting` proved it was unnecessary.

## AUR via yay (Arch-specific builder chain)

Unlike fedora-coder (which only uses `pixi`/`npm`/`cargo` builders), `arch-coder` can also pull from the AUR — because the `archlinux-builder` image ships `yay` and declares `builds: [pixi, npm, cargo, aur]`. Layers with `aur:` sections route through `archlinux-builder` as their `aur` builder. Currently no layer in the arch-coder stack uses AUR, but the capability exists for future additions (e.g. niche tools not in `[core]`/`[extra]`).

See `/ov-foundation:archlinux-builder` and `/ov-foundation:yay`.

## Cross-distro siblings

`arch-coder` is one of **four cross-distro coder images** that share the identical 80-line `tests:` block and ~30 identical layers, diverging only in each layer's package-format section (`rpm:` / `pac:` / `deb:`):

- `/ov-coder:fedora-coder` — RPM-based, Fedora 43 via `fedora-nonfree`, uid=1000 `user`.
- `/ov-coder:arch-coder` — this image; pacman + optional AUR, uid=1000 `user`.
- `/ov-coder:debian-coder` — deb-based, Debian 13 trixie, uid=1000 `user`.
- `/ov-coder:ubuntu-coder` — deb-based, Ubuntu 24.04 noble, uid=1000 `ubuntu` (adopted from base image — see `/ov-build:image` "user_policy" and `/ov-foundation:ubuntu`).

All four produce the same developer surface. Pick based on which distro family you (or your team / CI) are already standardized on.

## The layer stack

Layers in roughly the same groups as fedora-coder, with these Arch-specific notes:

- `nodejs` (not `nodejs24`) — Arch's `nodejs` package is already current enough; no LTS pinning.
- `language-runtimes` — Arch ships `dotnet-sdk` in `[extra]` (no Microsoft repo needed, unlike Debian/Ubuntu). Also drops `python3-ramalama` (not packaged on Arch — install via `uv tool install ramalama`).
- `dev-tools` — `/usr/bin/bat` ships directly from the `bat` package (no Debian-style batcat rename needed).
- `gh` — GitHub CLI from Arch's `extra` repo (no third-party apt repo needed).
- `container-nesting` — Arch equivalents for podman + buildah + skopeo + fuse-overlayfs + crun.

The remaining 25+ layers are identical to fedora-coder at the *composition* level. Each layer's `pac:` section handles the Arch-specific package names.

## Quick start

```bash
ov image build arch-coder
ov shell arch-coder
# or as a service:
ov config arch-coder
ov start arch-coder
ssh -p 2222 user@localhost           # passwordless sudo
```

## Verification recipe

```bash
ov image build arch-coder
ov eval image ghcr.io/overthinkos/arch-coder:latest
# disposable-container tests against the baked OCI label

ov config arch-coder
ov start arch-coder
ov eval image ghcr.io/overthinkos/arch-coder:latest
# adds deploy-scope checks: sshd reachable, supervisord, dbus, ov-mcp,
# virtqemud, libvirt session list, KVM domcaps, MCP tool-call probes.
```

## Ports

| Port | Service | Bound by |
|---|---|---|
| 2222 | sshd-wrapper (SSH access as `user` with sudo) | `/ov-coder:sshd` |
| 18765 | ov-mcp (entire `ov` CLI as MCP tools, Streamable HTTP) | `/ov-coder:ov-mcp` |

Conflicts with `/ov-coder:fedora-coder` / `/ov-coder:debian-coder` / `/ov-coder:ubuntu-coder` on both ports (all coder-family images use the same canonical ports). Use `-i <instance>` or `-p <remap>` to run alongside.

## Related images

- `/ov-foundation:archlinux` — base image (pacman bootstrap, `user_policy: auto` → create).
- `/ov-foundation:archlinux-builder` — pixi/npm/cargo/aur multi-stage builder.
- `/ov-coder:fedora-coder` — canonical RPM-family sibling.
- `/ov-coder:debian-coder` — deb-family sibling on Debian 13.
- `/ov-coder:ubuntu-coder` — deb-family sibling on Ubuntu 24.04 (adopt mode).
- `/ov-coder:arch-ov` — slimmer Arch alternative with just the ov toolchain (no AI CLIs or DevOps tooling).
- `/ov-selkies:selkies-desktop-ov` — adds a browser-streamed Wayland desktop on the fedora-ov baseline.

## Related layers

- `/ov-foundation:yay` — AUR helper, required for any `aur:` section
- `/ov-coder:ov-full`, `/ov-coder:ov-mcp`, `/ov-foundation:container-nesting` — shared rootless baseline
- `/ov-coder:sshd` — cross-distro sudoers via `getent passwd 1000`
- `/ov-coder:language-runtimes`, `/ov-coder:dev-tools`, `/ov-coder:gh`
- `/ov-coder:claude-code`, `/ov-coder:codex`, `/ov-coder:gemini`, `/ov-coder:forgecode`, `/ov-coder:oracle`

## Related commands

- `/ov-core:shell`, `/ov-core:config`, `/ov-core:start`, `/ov-core:stop`, `/ov-build:eval`
- `/ov-build:image` — `user_policy:` field reference
- `/ov-build:build` — `base_user:` declaration (absent for archlinux)
- `/ov-build:layer` — authoring reference

## When to use this skill

**MUST be invoked** when the task involves:

- Building, deploying, or troubleshooting `arch-coder`.
- Comparing cross-distro parity across the four coder-family images.
- Adding `aur:` sections to layers composed into `arch-coder`.
- Understanding why Arch doesn't need Microsoft's apt repo (dotnet-sdk is in `[extra]`) or batcat symlinks (`bat` is packaged as `bat`).
