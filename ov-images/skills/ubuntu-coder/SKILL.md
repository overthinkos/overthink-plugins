---
name: ubuntu-coder
description: |
  Kitchen-sink development image on Ubuntu 24.04 noble: coding + AI-coding
  CLIs + DevOps tooling in one container. Ubuntu base, 30+ direct layers
  mirroring fedora-coder's stack. Runs as uid 1000 `ubuntu` — the upstream
  ubuntu:24.04 account, adopted verbatim via build.yml's base_user
  declaration. 142/0/1-skip tests pass as of 2026-04-20.
  Use when working with the ubuntu-coder image — especially when the
  `${USER}` / `${HOME}` / sudoers differ from the other three coder images.
---

# ubuntu-coder

Ubuntu 24.04 noble counterpart of `/ov-images:fedora-coder`. Same 80-line test block, same ~30 layers, same rootless posture — but **the resolved user is `ubuntu` (not `user`)** because the upstream `ubuntu:24.04` base image ships a pre-existing `ubuntu:ubuntu` account at uid 1000, and `build.yml distro.ubuntu` declares `base_user:` to adopt it. Everything that touches the user account — `${HOME}`, npm prefix, pixi env, sudoers — derives from `resolved.User = "ubuntu"`.

## Definition

```yaml
ubuntu-coder:
  base: ubuntu
  ports:
    - "2222:2222"                 # sshd-wrapper
    - "18765:18765"               # ov-mcp (Streamable HTTP)
  layers:
    # Same stack as debian-coder — see that skill for the full list
    - agent-forwarding
    - sshd
    - ov-full
    - ov-mcp
    - container-nesting
    - dbus
    - tmux
    - language-runtimes
    - golang
    - nodejs
    - rust
    - pixi
    - uv
    - build-toolchain
    - dev-tools
    - pre-commit
    - direnv
    - gh
    - typst
    - asciinema
    - claude-code
    - codex
    - gemini
    - forgecode
    - oracle
    - devops-tools
    - kubernetes
    - docker-ce
    - github-actions
    - google-cloud
    - google-cloud-npm
    - grafana-tools
```

## User model — adopt mode

`build.yml distro.ubuntu.base_user` declares:

```yaml
base_user:
  name: ubuntu
  uid: 1000
  gid: 1000
  home: /home/ubuntu
```

`ubuntu-coder` doesn't set an explicit `user:` field, so `user_policy: auto` (the default) adopts the base_user. The generator emits no `useradd` — the bootstrap section is a one-line comment:

```
# User ubuntu (uid=1000) adopted from base image (declared in build.yml distro.base_user) — no useradd needed
WORKDIR /home/ubuntu
USER 1000
```

Resolved identity: `User=ubuntu, UID=1000, GID=1000, Home=/home/ubuntu`. The image's OCI labels carry these values (`org.overthinkos.user="ubuntu"`, `org.overthinkos.home="/home/ubuntu"`) so deploy-mode quadlets and `ov shell` find them without re-reading `image.yml`.

**Why adopt over rename?** Ubuntu's cloud-init tooling, documentation, and `/etc/passwd` metadata all expect the user to be named `ubuntu`. Renaming fights the upstream contract. Adopting honors it. See `/ov:image` "user_policy" for the full rationale and the 3-value policy table.

## Live verification (2026-04-20)

```
$ podman run --rm ghcr.io/overthinkos/ubuntu-coder:latest bash -c 'id; /usr/bin/dotnet --version; sudo -l'
uid=1000(ubuntu) gid=1000(ubuntu) groups=1000(ubuntu),4(adm),20(dialout),24(cdrom),25(floppy),27(sudo),29(audio),30(dip),44(video),46(plugdev),105(libvirt)
9.0.313
User ubuntu may run the following commands on <host>:
    (ALL : ALL) ALL
```

## Security posture

Identical to `/ov-images:debian-coder` — the only diff is `User` field.

| Field | Value | Source |
|---|---|---|
| `cap_add` | **(empty)** | — |
| `security_opt` | `[unmask=/proc/*]` | `/ov-layers:container-nesting` |
| `devices` | `[/dev/fuse, /dev/net/tun]` | `/ov-layers:container-nesting` |
| `privileged` | `false` | — |
| Network | bridge | defaults |
| UID / user | `1000 / ubuntu` | **adopt mode** |
| sudo | passwordless via `/etc/sudoers.d/ov-user` (discovered via `getent passwd 1000` → `ubuntu`) | `/ov-layers:sshd` |

## Ubuntu-specific layer quirks (differences vs debian-coder)

- **fastfetch test is skipped** via `exclude_distros: [ubuntu:24.04]` (`/ov-layers:dev-tools`). Ubuntu 24.04 noble main does not ship `fastfetch` (Debian 13 trixie does, and so do Fedora/Arch). Skipping is cleaner than failing — `exclude_distros:` is a 2026-04 addition to the declarative-test schema (see `/ov:test`).
- **dotnet-sdk-9.0 source**: uses Microsoft's `dotnet-install.sh` (same as debian-coder). Note: Canonical's noble main/universe ships `dotnet-sdk-8.0` and `dotnet-sdk-10.0` but NOT 9.0; Microsoft's noble apt repo ships 10.0 only. `dotnet-install.sh --channel 9.0` is the only cross-distro-consistent path to .NET 9. See `/ov-layers:language-runtimes`.
- **Everything else** is identical to debian-coder. The sudoers `getent` discovery, bat→batcat symlink, virtualization package-existence tests — all are implemented once in the respective layers and work uniformly on both debian-coder and ubuntu-coder.

## Empirical test results (2026-04-20)

`ov image test ghcr.io/overthinkos/ubuntu-coder:latest` — **142 passed · 0 failed · 1 skipped** (fastfetch, by design).

## Verification recipe

```bash
# 1. Validate
ov image validate

# 2. Build (auto-chains: ubuntu → ubuntu-builder → ubuntu-coder)
ov image build ubuntu-coder

# 3. Disposable-container tests
ov image test ghcr.io/overthinkos/ubuntu-coder:latest

# 4. Confirm adopt mode at runtime
podman run --rm ghcr.io/overthinkos/ubuntu-coder:latest id
# → uid=1000(ubuntu) gid=1000(ubuntu) ...

# 5. Deploy + live tests
ov config ubuntu-coder
ov start ubuntu-coder
ov image test ghcr.io/overthinkos/ubuntu-coder:latest --include-deploy
```

## Gotcha: Dockerhub rate-limits during base pulls

Rebuilding the `ubuntu` base image pulls from `docker.io/library/ubuntu:24.04`. Unauthenticated Dockerhub pulls are rate-limited (100/6h per IP). If the build fails with `toomanyrequests`, the cleanest workaround is to pull from a non-Dockerhub mirror and retag:

```bash
podman pull public.ecr.aws/docker/library/ubuntu:24.04
podman tag public.ecr.aws/docker/library/ubuntu:24.04 docker.io/library/ubuntu:24.04
ov image build ubuntu-coder
```

AWS ECR Public mirrors the Dockerhub library namespace without rate-limiting unauthenticated pulls.

## Ports

| Port | Service | Bound by |
|---|---|---|
| 2222 | sshd-wrapper (SSH as `ubuntu` with sudo) | `/ov-layers:sshd` |
| 18765 | ov-mcp | `/ov-layers:ov-mcp` |

Conflicts with the other three coder-family images on these ports.

## Image size

~10.2 GB uncompressed, nearly identical to `/ov-images:debian-coder`.

## Cross-distro siblings

- `/ov-images:fedora-coder` — RPM-family, `user:user` create.
- `/ov-images:arch-coder` — pacman-family + AUR, `user:user` create.
- `/ov-images:debian-coder` — deb on Debian 13, `user:user` create.
- `/ov-images:ubuntu-coder` — this image; deb on Ubuntu 24.04, `user:ubuntu` **adopt**.

## Related images

- `/ov-images:ubuntu` — parent base; declares `base_user:` in `build.yml`.
- `/ov-images:ubuntu-builder` — multi-stage builder, also runs as `ubuntu:1000`.

## Related layers

- `/ov-layers:sshd` — getent-based sudoers (works regardless of whether the uid-1000 account is `user` or `ubuntu`).
- `/ov-layers:dev-tools` — bat→batcat + fastfetch `exclude_distros`.
- `/ov-layers:language-runtimes` — Microsoft `dotnet-install.sh` for .NET 9.
- `/ov-layers:virtualization` — `package:` + `package_map:` tests.
- `/ov-layers:container-nesting`, `/ov-layers:ov-full`, `/ov-layers:ov-mcp` — shared rootless baseline.

## Related commands

- `/ov:image` — **`user_policy:` field** (the pivot for this image's behavior).
- `/ov:build` — **`base_user:` declaration** (where `ubuntu:1000` is declared).
- `/ov:test` — `exclude_distros:` field reference.
- `/ov:generate` — adopt-vs-create writeBootstrap emission.
- `/ov:shell`, `/ov:config`, `/ov:start`, `/ov:stop`.

## When to use this skill

**MUST be invoked** when:

- Building, deploying, or troubleshooting `ubuntu-coder`.
- Debugging any behavior that depends on `${USER}` or `${HOME}` inside an Ubuntu container (they resolve to `ubuntu` / `/home/ubuntu`, not `user` / `/home/user`).
- Writing a layer whose content references the uid-1000 account by name — it must not hardcode `user`; use `getent passwd 1000` or `${USER}` substitution in task fields where the generator expands it.
- Understanding Dockerhub rate-limit workarounds during base-image rebuilds.
