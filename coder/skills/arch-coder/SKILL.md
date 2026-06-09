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

> **Location:** the `arch-coder` image lives in the dedicated
> **`overthinkos/arch`** repo, mounted as a git submodule at **`image/arch`**.
> It composes the layers below by **git reference** to this repo
> (`@github.com/overthinkos/overthink/candy/<name>:<tag>`) rather than copying
> them; the `arch` base + `arch-builder` live in this repo (in `base.yml`) and
> are reached by the submodule importing this repo under the `charly` namespace
> (`base: charly.arch`, `builder: {…: charly.arch-builder}`). Build / deploy from
> the submodule, e.g. `cd image/arch && charly box build arch-coder` (the build
> verb defaults to the submodule's `charly.yml`), or
> `charly --repo overthinkos/arch image build arch-coder`. The commands below assume
> you are inside `image/arch` (or pass `-C image/arch`).

Arch Linux counterpart of `/charly-coder:fedora-coder`. Same daily-development surface (AI CLIs, language runtimes, DevOps stack), same rootless posture (uid 1000 + passwordless sudo), same shape of tests — only the package manager (`pac:` / `aur:`) and a handful of Arch-specific package names differ.

## Definition

```yaml
arch-coder:
  base: charly.arch                  # via the `charly` import namespace
  ports:
    - "2222:2222"                 # sshd-wrapper
    - "18765:18765"               # charly-mcp (Streamable HTTP)
  layers:
    # Baseline — identical to fedora-coder
    - agent-forwarding
    - sshd
    - charly
    - charly-mcp
    - container-nesting
    - dbus
    - tmux

    # Language runtimes + managers
    - language-runtimes           # Go + PHP + .NET 9 + python3-devel
    - golang
    - nodejs                      # Arch ships a current Node (v26)
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

No explicit `user:` / `uid:` / `gid:` — inherits the defaults. `user_policy:` defaults to `auto`; the `arch` base image declares no `base_user:` in `build.yml`, so the policy falls through to create mode — the bootstrap emits an idempotent `useradd -m -u 1000 -g 1000 user`. See `/charly-image:image` "user_policy" and `/charly-build:build` "base_user".

## Resolved security posture

| Field | Value | Source |
|---|---|---|
| `cap_add` | **(empty)** | — |
| `security_opt` | `[unmask=/proc/*]` | `/charly-distros:container-nesting` |
| `devices` | `[/dev/fuse, /dev/net/tun]` | `/charly-distros:container-nesting` |
| `privileged` | `false` | — |
| Network | bridge | defaults |
| UID / user | `1000 / user` | create mode |
| sudo | passwordless via `/etc/sudoers.d/charly-user` | `/charly-coder:sshd` |

**No `--privileged`, no `cap_add: ALL`, no `seccomp=unconfined`.** The rootless posture is shared with the three sibling power-user images (`fedora-coder`, `charly-fedora`, `charly-arch`); `/charly-distros:container-nesting` proves a privileged posture is unnecessary.

## AUR via yay (Arch-specific builder chain)

Unlike fedora-coder (which only uses `pixi`/`npm`/`cargo` builders), `arch-coder` can also pull from the AUR — because the `arch-builder` image ships `yay` and declares `builds: [pixi, npm, cargo, aur]`. Layers with `aur:` sections route through `arch-builder` as their `aur` builder. Currently no layer in the arch-coder stack uses AUR, but the capability exists for future additions (e.g. niche tools not in `[core]`/`[extra]`).

See `/charly-distros:arch-builder` and `/charly-tools:yay`.

## Cross-distro siblings

`arch-coder` is one of **four cross-distro coder images** that share the identical 80-line `eval:` block and ~30 identical layers, diverging only in each layer's package-format section (`rpm:` / `pac:` / `deb:`):

- `/charly-coder:fedora-coder` — RPM-based, Fedora 43 via `fedora-nonfree`, uid=1000 `user`.
- `/charly-coder:arch-coder` — this image; pacman + optional AUR, uid=1000 `user`.
- `/charly-coder:debian-coder` — deb-based, Debian 13 trixie, uid=1000 `user`.
- `/charly-coder:ubuntu-coder` — deb-based, Ubuntu 24.04 noble, uid=1000 `ubuntu` (adopted from base image — see `/charly-image:image` "user_policy" and `/charly-distros:ubuntu`).

All four produce the same developer surface. Pick based on which distro family you (or your team / CI) are already standardized on.

## The layer stack

Layers in roughly the same groups as fedora-coder, with these Arch-specific notes:

- `nodejs` — Arch's `nodejs` package ships a current Node (v26); no LTS pinning.
- `language-runtimes` — Arch ships `dotnet-sdk` in `[extra]` (no Microsoft repo needed, unlike Debian/Ubuntu). Also drops `python3-ramalama` (not packaged on Arch — install via `uv tool install ramalama`).
- `dev-tools` — `/usr/bin/bat` ships directly from the `bat` package (no Debian-style batcat rename needed).
- `gh` — GitHub CLI from Arch's `extra` repo (no third-party apt repo needed).
- `container-nesting` — Arch equivalents for podman + buildah + skopeo + fuse-overlayfs + crun.

The remaining 25+ layers are identical to fedora-coder at the *composition* level. Each layer's `pac:` section handles the Arch-specific package names.

## Quick start

```bash
charly box build arch-coder
charly shell arch-coder
# or as a service:
charly config arch-coder
charly start arch-coder
ssh -p 2222 user@localhost           # passwordless sudo
```

## Verification recipe

```bash
charly box build arch-coder
charly eval box ghcr.io/overthinkos/arch-coder:latest
# disposable-container tests against the baked OCI label

charly config arch-coder
charly start arch-coder
charly eval box ghcr.io/overthinkos/arch-coder:latest
# adds deploy-scope checks: sshd reachable, supervisord, dbus, charly-mcp,
# virtqemud, libvirt session list, KVM domcaps, MCP tool-call probes.
```

## Ports

| Port | Service | Bound by |
|---|---|---|
| 2222 | sshd-wrapper (SSH access as `user` with sudo) | `/charly-coder:sshd` |
| 18765 | charly-mcp (entire `charly` CLI as MCP tools, Streamable HTTP) | `/charly-coder:charly-mcp` |

Conflicts with `/charly-coder:fedora-coder` / `/charly-coder:debian-coder` / `/charly-coder:ubuntu-coder` on both ports (all coder-family images use the same canonical ports). Use `-i <instance>` or `-p <remap>` to run alongside.

## Related images

- `/charly-distros:arch` — base image (pacman bootstrap, `user_policy: auto` → create).
- `/charly-distros:arch-builder` — pixi/npm/cargo/aur multi-stage builder.
- `/charly-coder:fedora-coder` — canonical RPM-family sibling.
- `/charly-coder:debian-coder` — deb-family sibling on Debian 13.
- `/charly-coder:ubuntu-coder` — deb-family sibling on Ubuntu 24.04 (adopt mode).
- `/charly-coder:charly-arch` — slimmer Arch alternative with just the charly toolchain (no AI CLIs or DevOps tooling).
- `/charly-openclaw:openclaw-desktop` — adds a browser-streamed Wayland desktop (plus the openclaw gateway, AI CLIs, and a CPU ollama) with the same rootless charly toolchain.

## Related layers

- `/charly-tools:yay` — AUR helper, required for any `aur:` section
- `/charly-tools:charly`, `/charly-coder:charly-mcp`, `/charly-distros:container-nesting` — shared rootless baseline
- `/charly-coder:sshd` — cross-distro sudoers via `getent passwd 1000`
- `/charly-coder:language-runtimes`, `/charly-coder:dev-tools`, `/charly-coder:gh`
- `/charly-coder:claude-code`, `/charly-coder:codex`, `/charly-coder:gemini`, `/charly-coder:forgecode`, `/charly-coder:oracle`

## Related commands

- `/charly-core:shell`, `/charly-core:charly-config`, `/charly-core:start`, `/charly-core:stop`, `/charly-eval:eval`
- `/charly-image:image` — `user_policy:` field reference
- `/charly-build:build` — `base_user:` declaration (absent for arch)
- `/charly-image:layer` — authoring reference

## When to use this skill

**MUST be invoked** when the task involves:

- Building, deploying, or troubleshooting `arch-coder`.
- Comparing cross-distro parity across the four coder-family images.
- Adding `aur:` sections to layers composed into `arch-coder`.
- Understanding why Arch doesn't need Microsoft's apt repo (dotnet-sdk is in `[extra]`) or batcat symlinks (`bat` is packaged as `bat`).
