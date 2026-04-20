---
name: fedora-coder
description: |
  Kitchen-sink development image: coding + AI-coding CLIs + DevOps tooling
  in one container. Fedora-nonfree base, 32 direct layers spanning language
  runtimes, build tooling, five AI coding CLIs, and the full cloud/devops
  stack. Runs as uid 1000 with passwordless sudo — rootless-first, matches
  the /ov-images:selkies-desktop-ov security posture.
  Use when working with the fedora-coder image — specifically any task that
  involves SSH-ing into a single container and having every tool a polyglot
  engineer reaches for during a working day already installed.
---

# fedora-coder

The everyday-development counterpart to `/ov-images:fedora-ov`: all the
coding + AI + DevOps tooling a developer needs in one image, no desktop,
no streaming. Distinct from `selkies-desktop-ov` (which adds the
browser-streamed Wayland desktop) — `fedora-coder` is headless and
meant to be accessed via `ssh -p 2222` or `ov shell`.

## Definition

```yaml
fedora-coder:
  base: fedora-nonfree            # = fedora + rpmfusion (for x264/ffmpeg-devel)
  ports:
    - "2222:2222"                 # sshd-wrapper
    - "18765:18765"               # ov-mcp (Streamable HTTP)
  layers:
    # Baseline (matches the ov-ov power-user pattern)
    - agent-forwarding
    - sshd
    - ov-full                     # ov + virtualization + gocryptfs + socat
    - ov-mcp                      # MCP gateway for the entire ov CLI
    - container-nesting           # rootless nested podman/buildah/skopeo
    - dbus
    - tmux

    # Language runtimes + managers
    - language-runtimes           # Go + PHP + .NET 9 + nodejs-devel + python3-devel
    - golang
    - nodejs24
    - rust
    - pixi
    - uv                          # direct-download binary (no pixi env)

    # Build + developer tooling
    - build-toolchain
    - dev-tools                   # ripgrep, bat, fd, neovim, htop, …
    - pre-commit
    - direnv
    - gh                          # GitHub CLI + git + git-lfs (single-responsibility)
    - typst
    - asciinema

    # AI coding CLIs
    - claude-code
    - codex
    - gemini
    - forgecode                   # 5th CLI (astral-sh-style)
    - oracle

    # DevOps / cloud / infra
    - devops-tools                # AWS, Scaleway, kubectx/kubens, OpenTofu
    - kubernetes                  # kubectl, helm, k3d
    - docker-ce
    - github-actions              # act, actionlint
    - google-cloud
    - google-cloud-npm            # firebase + Gemini npm bindings
    - grafana-tools
```

Network: default `ov` bridge. `network: host` was dropped 2026-04 once the
sibling `/ov-images:arch-coder` landed — the `ports:` mapping is the
right way to expose sshd/ov-mcp, and bridge networking lets dev boxes
coexist on one host via normal `-p host:container` remapping (e.g.
running `arch-coder` alongside `fedora-coder` on the same machine).

No `uid:` / `gid:` / `user:` / `security:` override — inherits
`1000/1000/user` from `defaults` and the layer-level security from
`/ov-layers:container-nesting`. Rootless-first by design (shares this
posture with `/ov-images:selkies-desktop-ov`).

## Resolved security posture (OCI label)

| Field | Value | Source |
|---|---|---|
| `cap_add` | **(empty)** | — |
| `security_opt` | `[unmask=/proc/*]` | `/ov-layers:container-nesting` |
| `devices` | `[/dev/fuse, /dev/net/tun]` | `/ov-layers:container-nesting` |
| `privileged` | `false` | — |
| Network | **host** (host namespace) | image.yml |
| UID / user | `1000 / user` | defaults |
| sudo | passwordless via `/etc/sudoers.d/ov-user` | `/ov-layers:sshd` |

**No `--privileged`. No `cap_add: ALL`. No `seccomp=unconfined`. No
`label=disable`.** All four outliers (`fedora-coder`, `fedora-ov`,
`arch-ov`, `githubrunner`) dropped the legacy root posture in 2026-04
once it was proven redundant by the `container-nesting` kernel RCA —
see `/ov-layers:container-nesting` for the `mount_too_revealing()` +
`unmask=/proc/*` derivation.

## Network posture

Default `ov` bridge. `network: host` was dropped 2026-04 — it prevented
running `fedora-coder` alongside `/ov-images:arch-coder` on the same host
(both want sshd on 2222 and ov-mcp on 18765). Bridge networking plus
normal `-p host:container` remapping is the right pattern for dev boxes.
To run two dev images side by side:

```bash
ov config arch-coder -p 2223:2222 -p 18766:18765     # alt-instance ports
ov config fedora-coder                               # canonical 2222/18765
```

Reaching host services from inside a bridge-networked container: use
the host's LAN IP or `host.containers.internal` (podman) rather than
`127.0.0.1`. For selkies-style sibling-container discovery, the deploy.yml
`provides.env:` and `OV_MCP_SERVERS` auto-injection give explicit hostnames.

## The 32-layer composition explained

**Why `fedora-nonfree` (not plain `fedora`)** — `build-toolchain` pulls
`x264-devel` + `ffmpeg-devel`, both of which live in RPM Fusion. Every
image that uses `build-toolchain` either bases on `fedora-nonfree` or
includes `rpmfusion` explicitly.

**Python story** — `python` (the pixi-python ov-layer) is NOT pulled in.
As of 2026-04, `supervisord`, `language-runtimes`, and `uv` all dropped
their vestigial `depends: python` (they use system python3 from RPM, not
the conda-forge pixi env). System Python is available via
`language-runtimes` (`python3-devel` + `python3-ramalama`). See the
"Key Rules" note in `CLAUDE.md` ("don't declare defensive deps").

**`uv` is a direct-download binary** now (no pixi involvement). Lives at
`/usr/local/bin/uv` and `/usr/local/bin/uvx`, extracted from the
upstream astral-sh/uv tarball via the new `download:` verb
`strip_components: 1` modifier. See `/ov-layers:uv` and `/ov:layer`.

**`gh` owns all git tooling** — the `gh` layer installs `gh` + `git` +
`git-lfs` (with the noscripts + post-install trigger). `dev-tools`
intentionally does NOT install any of these. See `/ov-layers:gh`.

**No GPU by default** — `nvidia` / `cuda` intentionally NOT composed. If
you need CUDA in your dev image, spin a sibling `fedora-coder-nvidia`
with `base: nvidia` + the same layer list.

## Verification recipe (build + test + deploy-test)

```bash
# 1. Validate config
ov image validate

# 2. Build (first time ≈ 15–30 min; cached rebuilds are fast thanks to
# LABEL-at-end ordering — test edits rebuild in seconds)
ov image build fedora-coder

# 3. Build-scope tests (disposable container)
ov image test ghcr.io/overthinkos/fedora-coder:latest
# target: 149 passed · 0 failed · 0 skipped

# 4. Deploy (no bind-mount needed; ov-mcp auto-falls back to overthinkos/overthink)
ov config fedora-coder
ov start fedora-coder

# 5. Full-scope tests — prefers the running container automatically
# (see /ov:test "Live vs disposable executor selection")
ov image test ghcr.io/overthinkos/fedora-coder:latest --include-deploy
# target: 167 passed · 0 failed · 0 skipped

# 6. Clean up
ov stop fedora-coder
```

If you want `image.list.images` over the MCP surface to list YOUR
local checkout's images (rather than upstream overthinkos/overthink),
bind-mount the project:

```bash
ov config fedora-coder --bind project=/home/you/overthink
ov start fedora-coder
```

The volume NAME is `project` (stable bind-mount API); the in-container
mount path is `/workspace`. See `/ov-layers:ov-mcp` for the three
deployment patterns (bind-mount / `OV_PROJECT_REPO` / auto-fallback).

## Ports

| Port | Service | Bound by |
|---|---|---|
| 2222 | sshd-wrapper (SSH access as user with sudo) | `/ov-layers:sshd` |
| 18765 | ov-mcp (entire `ov` CLI as MCP tools, Streamable HTTP) | `/ov-layers:ov-mcp` |

Conflicts with `/ov-images:arch-ov` (identical ports) and several
selkies-desktop variants on 2222. Use `-i <instance>` or `-p <remap>`
if running alongside.

## Empirical test results (2026-04-20)

- `ov image test fedora-coder` — **149 passed · 0 failed · 0 skipped**.
- `ov image test fedora-coder --include-deploy` against a live running
  container — **167 passed · 0 failed · 0 skipped** (adds deploy-scope:
  sshd reachable, supervisord responding, dbus+ov-mcp+virtqemud+
  virtnetworkd services running, libvirt session list + KVM domcaps,
  mcp ping/list-tools/call-version/call-list-images via the ov-mcp
  auto-fallback).

## Image size

~14 GB uncompressed (kitchen-sink trade-off). Dominated by:

- `virtualization` — QEMU + libvirt (~800 MB)
- `docker-ce` — Docker engine + containerd + buildx + compose (~500 MB)
- 5 AI coding CLIs — `forgecode` alone is ~370 MB (native binary); the
  JS-only ones are smaller.
- `grafana-tools` — 7 binaries fetched from GitHub releases (~200 MB)
- `language-runtimes` — .NET 9 SDK is ~600 MB on its own.

To slim: drop the layer groups you don't need (AI CLIs, kubernetes,
google-cloud-npm) by forking image.yml. See `/ov:image` for authoring.

## Key Layers

- `/ov-layers:ov-full` — composition: ov + virtualization + gocryptfs + socat (+ podman-machine, gvisor-tap-vsock for nested VMs)
- `/ov-layers:ov-mcp` — MCP gateway; auto-falls back to `overthinkos/overthink` when no bind-mount present
- `/ov-layers:container-nesting` — rootless nested podman recipe (authoritative RCA for `mount_too_revealing()` + `unmask=/proc/*`)
- `/ov-layers:sshd` — SSH daemon + passwordless sudo for the `user` account
- `/ov-layers:forgecode` — the 5th AI coding CLI (new in 2026-04)
- `/ov-layers:language-runtimes` — Go + PHP + .NET + nodejs-devel + python3-devel (system-python stack, no pixi)
- `/ov-layers:uv` — direct-download binary at /usr/local/bin/uv (rewritten 2026-04)
- `/ov-layers:gh` — GitHub CLI + git + git-lfs (single-responsibility, 2026-04)

## Cross-distro siblings

`fedora-coder` is one of **four cross-distro coder images** that share the identical 80-line `tests:` block + ~30 identical layers; they diverge only in each layer's package-format section (`rpm:` / `pac:` / `deb:`) and a handful of distro-specific quirks handled inside individual layers.

| Image | Base | Package mgr | User model |
|---|---|---|---|
| `/ov-images:fedora-coder` | `fedora-nonfree` (Fedora 43) | rpm | `user:user` (create) |
| `/ov-images:arch-coder` | `archlinux` | pac + aur | `user:user` (create) |
| `/ov-images:debian-coder` | `debian:13` | deb | `user:user` (create) |
| `/ov-images:ubuntu-coder` | `ubuntu:24.04` | deb | `user:ubuntu` (**adopt** from base) |

All four produce the same daily-dev surface (sshd on 2222, ov-mcp on 18765, 5 AI CLIs, full language runtimes, DevOps tooling). Pick based on distro-family alignment with your team/CI. See `/ov:image` "user_policy" for the adopt-vs-create reconciliation that gives ubuntu-coder its `ubuntu` username.

## Related Images

- `/ov-images:selkies-desktop-ov` — sibling rootless-first power-user image; same security posture + container-nesting, but adds the streaming desktop. Prefer when you want browser-accessible GUI + dev tools.
- `/ov-images:fedora-ov` — minimal ov toolchain (no coding CLIs, no DevOps), also uid=1000 with sudo.
- `/ov-images:arch-ov` — Arch Linux counterpart of fedora-ov.
- `/ov-images:githubrunner` — self-hosted GitHub Actions runner; same uid=1000 posture.
- `/ov-images:hermes` — adds the Hermes AI agent daemon on top of the AI CLIs; prefer when you want an agent, not just CLI tools.

## Related Commands

- `/ov:shell` — open an interactive shell inside the container (as user, with sudo)
- `/ov:config` — deploy setup (--bind project=…, tunnel, --update-all)
- `/ov:start`, `/ov:stop` — lifecycle
- `/ov:test` — live-service and build-scope tests (`--include-deploy` prefers running container)
- `/ov:mcp` — MCP gateway documentation; `--no-default-repo` to disable auto-fallback
- `/ov:vm` — nested libvirt VMs (via virtqemud inside the container)
- `/ov:layer` — authoring reference (covers the new `strip_components:` modifier)

## When to Use This Skill

**MUST be invoked** when the task involves:

- Building, deploying, or troubleshooting the `fedora-coder` image.
- Picking the right power-user base image for a coding/dev workload (this
  vs. `fedora-ov` vs. `arch-ov` vs. `selkies-desktop-ov`).
- Understanding the rootless-first architectural pattern shared by the
  four power-user images (kernel RCA belongs in
  `/ov-layers:container-nesting`; the composition that proves it works
  without a GUI is here).
- The full MCP auto-fallback deployment pattern when the `/workspace`
  bind-mount is intentionally left empty.
