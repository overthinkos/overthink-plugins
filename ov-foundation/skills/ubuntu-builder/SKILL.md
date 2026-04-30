---
name: ubuntu-builder
description: |
  Minimal Ubuntu 24.04 builder image (pixi + Node.js + build-toolchain)
  used as the multi-stage builder for Ubuntu-based images — currently
  ubuntu-coder. Runs as uid 1000 `ubuntu` (adopted from the upstream
  ubuntu:24.04 base image via build.yml's base_user declaration).
  MUST be invoked before building, deploying, configuring, or troubleshooting
  the ubuntu-builder image.
---

# ubuntu-builder

Ubuntu 24.04 (noble) counterpart of `/ov-foundation:fedora-builder` and `/ov-foundation:debian-builder`. Same role — pixi/npm/cargo multi-stage builder — with one important difference: the builder runs as `ubuntu` (uid 1000) because the upstream `ubuntu:24.04` base image ships a pre-existing `ubuntu:ubuntu` account at uid 1000, and `build.yml distro.ubuntu.base_user` adopts it.

## Image Properties

| Property | Value |
|----------|-------|
| Base | `ubuntu` (which = `ubuntu:24.04` + our bootstrap) |
| Layers | `pixi`, `nodejs`, `build-toolchain` |
| Platforms | `linux/amd64` |
| Registry | `ghcr.io/overthinkos` |
| User | `ubuntu` / uid 1000 (**adopt mode** — see `/ov-build:image` "user_policy") |
| Home | `/home/ubuntu` |

## Full layer stack

1. `/ov-foundation:ubuntu` — Ubuntu 24.04 + bootstrap. Inherits Debian's `apt-get update && apt-get install -y --no-install-recommends curl ca-certificates gnupg` pattern because `build.yml distro.ubuntu` declares `inherits: debian`. Ubuntu-specific: `base_user: { name: ubuntu, uid: 1000, gid: 1000, home: /home/ubuntu }` — no `useradd` step emitted.
2. `/ov-foundation:pixi` — pixi package manager + env paths (`/home/ubuntu/.pixi`).
3. `/ov-coder:nodejs` — Node.js + npm (generic `nodejs`, not `nodejs24`).
4. `/ov-coder:build-toolchain` — same Debian `-dev` packages as `/ov-foundation:debian-builder`.

## Adopt-mode semantics

When the generator emits the Containerfile for this image, the bootstrap section contains:

```
# User ubuntu (uid=1000) adopted from base image (declared in build.yml distro.base_user) — no useradd needed
WORKDIR /home/ubuntu
USER 1000
```

No `useradd`, no `groupadd`, no `usermod -l` rename. The upstream `ubuntu:ubuntu` account is honored verbatim — HOME, npm prefix, pixi env, cargo home, sudoers all derive from `resolved.User = "ubuntu"`. See `/ov-build:image` "user_policy" and `/ov-dev:generate` "writeBootstrap".

## Role in the build system

Declares `builds: [pixi, npm, cargo]` and is referenced from `ubuntu:` as:

```yaml
ubuntu:
  builder:
    pixi: ubuntu-builder
    npm: ubuntu-builder
    cargo: ubuntu-builder
```

During `ov image build ubuntu-coder`, cargo/npm/pixi-owning layers get their `FROM ubuntu-builder AS <layer>-<type>-build` stages from this image, then `COPY --from=<stage> --chown=1000:1000 /home/ubuntu /home/ubuntu` into the final ubuntu-coder. (The `--chown=1000:1000` numeric form works uniformly regardless of user name — see `/ov-coder:build-toolchain` for the builder-artifact COPY pattern.)

## Cross-distro sibling builders

- `/ov-foundation:fedora-builder` — RPM-family, `user:user` uid 1000 (create).
- `/ov-foundation:debian-builder` — deb-family, Debian 13, `user:user` (create — Debian 13 ships no pre-existing uid-1000 user).
- `/ov-foundation:archlinux-builder` — pacman-family, `user:user` + `yay` for AUR.

The three builders have near-identical layer stacks (pixi + nodejs + build-toolchain). The only meaningful divergence is this image's adopt-mode `ubuntu:ubuntu` identity.

## Quick start

```bash
ov image build ubuntu-builder
ov shell ubuntu-builder               # drops you into /home/ubuntu as uid 1000
id                                    # uid=1000(ubuntu) gid=1000(ubuntu)
```

Typically not invoked directly — it's a build-time dependency of `/ov-coder:ubuntu-coder`.

## Verification

- `ov image list | grep ubuntu-builder`
- `ov shell ubuntu-builder -- id` → `uid=1000(ubuntu) gid=1000(ubuntu)`
- `ov shell ubuntu-builder -- pixi --version && node --version && gcc --version`

## Related images

- `/ov-foundation:ubuntu` — parent base; declares `base_user:` in `build.yml`.
- `/ov-coder:ubuntu-coder` — the consumer that this builder serves.
- `/ov-foundation:debian-builder` — deb-family sibling without adopt mode.
- `/ov-foundation:fedora-builder` — canonical RPM-family sibling.

## Related layers

- `/ov-foundation:pixi`, `/ov-coder:nodejs`, `/ov-coder:build-toolchain`

## Related commands

- `/ov-build:build` — `base_user:` declaration format in `build.yml distro.*`
- `/ov-build:image` — `user_policy:` field (auto / adopt / create) and the decision table
- `/ov-build:generate` — adopt-vs-create writeBootstrap emission modes

## When to use this skill

**MUST be invoked** when:

- Building or troubleshooting `ubuntu-builder` itself.
- Debugging multi-stage COPY-from errors or permission issues in a `ubuntu-coder` build (this is the source stage; uid 1000 = `ubuntu` by adoption).
- Understanding why `${HOME}` = `/home/ubuntu` (not `/home/user`) inside builder stages.
