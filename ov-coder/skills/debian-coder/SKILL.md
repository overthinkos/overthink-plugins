---
name: debian-coder
description: |
  Kitchen-sink development image on Debian 13 trixie: coding + AI-coding
  CLIs + DevOps tooling in one container. Debian base, 30+ direct layers
  mirroring fedora-coder's stack but with deb: sections. Runs as uid 1000
  (`user`) with passwordless sudo. 143/0 tests pass as of 2026-04-20.
  Use when working with the debian-coder image — or when comparing
  cross-distro parity across the four coder-family images.
---

# debian-coder

Debian 13 trixie counterpart of `/ov-coder:fedora-coder`. Same 80-line `tests:` block, same ~30 layers, same rootless posture (uid 1000 + passwordless sudo). Key wrinkles are all Debian-specific packaging quirks handled inside individual layers: `bat → batcat` symlink, Microsoft's `dotnet-install.sh` cross-distro installer, and package-existence tests (vs binary-path tests) for `virtualization` because Debian bundles libvirt drivers differently.

## Definition

```yaml
debian-coder:
  base: debian
  ports:
    - "2222:2222"                 # sshd-wrapper
    - "18765:18765"               # ov-mcp (Streamable HTTP)
  layers:
    # Baseline
    - agent-forwarding
    - sshd
    - ov-full
    - ov-mcp
    - container-nesting
    - dbus
    - tmux

    # Language runtimes
    - language-runtimes           # Go + PHP + .NET 9 (via Microsoft dotnet-install.sh)
    - golang
    - nodejs                      # plain nodejs — Debian ships no nodejs24
    - rust
    - pixi
    - uv

    # Build + developer tooling
    - build-toolchain
    - dev-tools                   # ripgrep, bat→batcat symlink, fd-find, neovim, …
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

## User model — create mode

The `debian:13` upstream base image ships **no** pre-existing uid-1000 account. `build.yml distro.debian` declares no `base_user:`, so `user_policy: auto` (the default) falls through to create mode. The bootstrap section of `.build/debian/Containerfile` emits:

```
RUN if ! getent passwd 1000 >/dev/null 2>&1; then
      (getent group 1000 >/dev/null 2>&1 || groupadd -g 1000 user) &&
      useradd -m -u 1000 -g 1000 -s /bin/bash user;
    fi
```

Resolved identity: `User=user, UID=1000, GID=1000, Home=/home/user`. Identical to fedora-coder / arch-coder. Contrast with `/ov-coder:ubuntu-coder` which runs adopt mode on `ubuntu:ubuntu`.

See `/ov-build:image` "user_policy" and `/ov-build:build` "base_user" for the full decision table.

## Security posture

| Field | Value | Source |
|---|---|---|
| `cap_add` | **(empty)** | — |
| `security_opt` | `[unmask=/proc/*]` | `/ov-foundation:container-nesting` |
| `devices` | `[/dev/fuse, /dev/net/tun]` | `/ov-foundation:container-nesting` |
| `privileged` | `false` | — |
| Network | bridge | defaults |
| UID / user | `1000 / user` | create mode |
| sudo | passwordless via `/etc/sudoers.d/ov-user` (discovered via `getent passwd 1000`) | `/ov-coder:sshd` |

## Debian-specific layer quirks

- **bat → batcat symlink** (`/ov-coder:dev-tools`). Debian and Ubuntu rename `bat` → `batcat` to avoid a namespace collision with a legacy `bacula` utility. The dev-tools layer ships a distro-tolerant cmd task that creates `/usr/bin/bat -> /usr/bin/batcat` when only batcat is present; no-op on Fedora/Arch.
- **dotnet-sdk-9.0 via Microsoft's `dotnet-install.sh`** (`/ov-coder:language-runtimes`). Debian 13 main ships no .NET; Microsoft's trixie apt repo has dotnet-sdk-9.0, but using the official cross-distro `dotnet-install.sh` channel-pin to `9.0` gives version parity with Ubuntu (whose Microsoft noble repo only ships 10.0). Installs to `/usr/share/dotnet` + symlinks `/usr/bin/dotnet`.
- **libvirt tests use `package:` not `file:`** (`/ov-foundation:virtualization`). Debian bundles libvirt drivers differently from Fedora's split packaging, so `file: /usr/sbin/virtqemud` would false-fail. The layer now probes via `package: libvirt-daemon-driver-qemu` + `package_map:` per-distro package name.
- **sudoers via `getent`** (`/ov-coder:sshd`). Instead of hardcoding `user ALL=…`, the layer runs `getent passwd 1000 | cut -d: -f1` at build time to discover the actual uid-1000 account. On debian-coder that returns `user`; on ubuntu-coder it returns `ubuntu`. One cross-distro implementation, no special cases.

## Empirical test results (2026-04-20)

`ov eval image ghcr.io/overthinkos/debian-coder:latest` — **143 passed · 0 failed · 0 skipped**.

`ov eval image ghcr.io/overthinkos/debian-coder:latest` — live-service extension tests (sshd on 2222, supervisord, dbus, ov-mcp, virtqemud session) — not run in Phase F (disk-constrained CI environment), but expected to mirror fedora-coder's +18 additions.

## Verification recipe

```bash
# 1. Validate
ov image validate

# 2. Build dependencies auto-resolve (debian → debian-builder → debian-coder)
ov image build debian-coder

# 3. Disposable-container tests
ov eval image ghcr.io/overthinkos/debian-coder:latest

# 4. Deploy + live tests
ov config debian-coder
ov start debian-coder
ov eval image ghcr.io/overthinkos/debian-coder:latest

# 5. Clean up
ov stop debian-coder
```

## Ports

| Port | Service | Bound by |
|---|---|---|
| 2222 | sshd-wrapper (SSH access as `user` with sudo) | `/ov-coder:sshd` |
| 18765 | ov-mcp (entire `ov` CLI as MCP tools, Streamable HTTP) | `/ov-coder:ov-mcp` |

Conflicts with the other three coder-family images on the same ports — use `-i <instance>` or `-p <remap>` to run alongside.

## Image size

~10.4 GB uncompressed. Comparable to fedora-coder (~14 GB); smaller due to tighter Debian package naming (no split `-devel` packages, sparser optional dependencies pulled by default).

## Cross-distro siblings

All four coder-family images share the identical 80-line `tests:` block + ~30 identical layers; they diverge only in per-layer package-format sections.

- `/ov-coder:fedora-coder` — RPM (Fedora 43 via fedora-nonfree).
- `/ov-coder:arch-coder` — pacman + optional AUR.
- `/ov-coder:ubuntu-coder` — deb on Ubuntu 24.04; **adopt mode** (`user:ubuntu`).
- `/ov-coder:debian-coder` — this image; create mode (`user:user`).

## Related images

- `/ov-foundation:debian` — parent base.
- `/ov-foundation:debian-builder` — multi-stage builder for pixi/npm/cargo.

## Related layers (Debian-specific content)

- `/ov-coder:sshd` — cross-distro sudoers via `getent passwd 1000`.
- `/ov-coder:dev-tools` — bat → batcat symlink task.
- `/ov-coder:language-runtimes` — Microsoft `dotnet-install.sh` cross-distro pattern.
- `/ov-foundation:virtualization` — `package:` + `package_map:` pattern for libvirt.
- `/ov-coder:build-toolchain` — Debian `-dev` package equivalents of Fedora's `-devel`.
- `/ov-coder:gh`, `/ov-coder:docker-ce`, `/ov-coder:kubernetes-layer`, `/ov-foundation:container-nesting` — each adds an upstream apt repo (with GPG key) for packages not in Debian main.

## Related commands

- `/ov-core:shell`, `/ov-core:config`, `/ov-core:start`, `/ov-core:stop`, `/ov-build:eval`
- `/ov-build:image` — `user_policy:` field reference
- `/ov-build:build` — `base_user:` declaration (absent for Debian)
- `/ov-build:layer` — authoring reference (covers `exclude_distros:`, tag-section `repos:`, Microsoft dotnet-install pattern)

## When to use this skill

**MUST be invoked** when the task involves:

- Building, deploying, or troubleshooting `debian-coder`.
- Comparing cross-distro parity, especially between `debian-coder` and `ubuntu-coder` (the base images differ in their pre-existing user account).
- Understanding why Debian needs the `dotnet-install.sh` shell task (vs Fedora's native `dotnet-sdk` rpm).
- Adding new deb: sections to layers that are currently Fedora/Arch-only.
