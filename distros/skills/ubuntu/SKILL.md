---
name: ubuntu
description: |
  Base Ubuntu 24.04 noble image. Root of the image hierarchy for Ubuntu-
  based builds. Runs as uid 1000 `ubuntu` via ADOPT mode — the upstream
  ubuntu:24.04 base image ships a pre-existing ubuntu:ubuntu account,
  and build.yml distro.ubuntu declares base_user to adopt it verbatim.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  any Ubuntu-based image.
---

# ubuntu

Base Ubuntu 24.04 (noble) image. Distinguished from `/charly-distros:debian` by **adopt mode**: the upstream `ubuntu:24.04` base image ships a pre-existing `ubuntu:ubuntu` account at uid 1000, and `build.yml distro.ubuntu` declares `base_user:` so the `ov` generator honors that account rather than creating a new one.

The Ubuntu family lives in its own **`overthinkos/ubuntu`** repo (git submodule
at **`image/ubuntu`**) — a SEPARATE repo from `overthinkos/debian` (Debian and
Ubuntu each have their own repo). The `ubuntu` base is **owned there** and
composes the main repo's layers + shared `build.yml` by git reference. Because
`distro.ubuntu` is `inherits: debian`, the single remote `build.yml` (which
carries BOTH distro configs) resolves the inheritance — `overthinkos/ubuntu`
needs no reference to `overthinkos/debian`. Build from the submodule:
`charly -C image/ubuntu image build ubuntu` (or `charly --repo overthinkos/ubuntu image build ubuntu`).
Nothing in main consumes any Ubuntu image, so there is **no main ↔ ubuntu coupling**.

## Image Properties

| Property | Value |
|----------|-------|
| Base | `ubuntu:24.04` |
| Pkg | `deb` |
| Distro tags | `["ubuntu:24.04", "ubuntu"]` (symmetric with a `target: vm` deploy's `distroTagChain`; NO `debian` fallback — every deb-family layer carries an explicit `ubuntu` section, and the cascade would otherwise UNION debian-only packages onto ubuntu) |
| Layers | (none — base image only) |
| Platforms | `linux/amd64` |
| User | `ubuntu` / uid 1000 (**adopt mode**) |
| Home | `/home/ubuntu` |
| Registry | `ghcr.io/overthinkos` |

## User model — adopt from base image

`build.yml distro.ubuntu` inherits from `distro.debian` (same apt bootstrap template) and adds a `base_user:` block:

```yaml
distro:
  ubuntu:
    inherits: debian
    base_user:
      name: ubuntu
      uid: 1000
      gid: 1000
      home: /home/ubuntu
```

Any downstream image with `user_policy: auto` (the default) that **did not** explicitly set its own `user:` field will adopt this — `resolved.User = "ubuntu"`, `resolved.Home = "/home/ubuntu"`, `resolved.UserAdopted = true`. The bootstrap emits **no** `useradd`; it emits a one-line comment documenting the adoption:

```
# User ubuntu (uid=1000) adopted from base image (declared in build.yml distro.base_user) — no useradd needed
WORKDIR /home/ubuntu
USER 1000
```

This architecture is **declarative** (what the base image ships) + **policy-driven** (how to reconcile with the image's configured user). Three policy values:

| Policy | Behavior on this base |
|--------|----------------------|
| `auto` (default) | Adopt `ubuntu:ubuntu` — image inherits the upstream account. |
| `adopt` | Same as auto here; hard-errors on bases without `base_user:`. |
| `create` | Override — force-create a different uid-1000 account (fails if `useradd` collides). |

See `/charly-image:image` "user_policy" and `/charly-build:build` "base_user" for the full table covering all four distros.

## Why adopt over rename?

Adopt mode honors the existing `ubuntu` account rather than renaming it to `user` via `usermod -l`, because:

1. Ubuntu's cloud-init tooling, docs, and `/etc/passwd` metadata assume the account is named `ubuntu`.
2. Renaming is an invisible base-image mutation — breaks in hard-to-debug ways.
3. The rename approach doesn't scale to Debian cloud images (or future distros) that ship their own pre-existing uid-1000 accounts with different names.

Adopt mode respects the base image's contract and scales declaratively. See `/charly-coder:sshd` for the `getent passwd 1000` pattern that makes layer content (sudoers in particular) work uniformly across both create and adopt modes.

## Bootstrap

```dockerfile
FROM ubuntu:24.04
RUN --mount=type=cache,dst=/var/cache/apt,sharing=locked
    --mount=type=cache,dst=/var/lib/apt,sharing=locked
    apt-get update && apt-get install -y --no-install-recommends curl ca-certificates gnupg && \
    ... install go-task binary ...
# User ubuntu (uid=1000) adopted from base image (declared in build.yml distro.base_user) — no useradd needed
WORKDIR /home/ubuntu
USER 1000
```

## Dockerhub rate-limit caveat

The upstream `ubuntu:24.04` pull from Dockerhub is unauthenticated-rate-limited (100 pulls / 6h / IP). If `charly box build ubuntu` fails with `toomanyrequests`, pull from AWS ECR Public and retag:

```bash
podman pull public.ecr.aws/docker/library/ubuntu:24.04
podman tag public.ecr.aws/docker/library/ubuntu:24.04 docker.io/library/ubuntu:24.04
charly box build ubuntu
```

ECR Public mirrors the Dockerhub library namespace without rate-limiting.

## Downstream / sibling entries (all in overthinkos/ubuntu)

- `/charly-distros:ubuntu-builder` — pixi/npm/cargo multi-stage builder.
- `/charly-coder:ubuntu-coder` — kitchen-sink dev image.
- `/charly-distros:ubuntu-debootstrap-builder` — privileged debootstrap builder (`base: debian:13`).
- `/charly-distros:ubuntu-debootstrap` — bootstrap-from-scratch rootfs.
- `/charly-vm:ubuntu` — the `ubuntu-debootstrap` bootstrap VM + `eval-ubuntu-debootstrap-vm` bed.

## Verification

```bash
charly -C image/ubuntu image build ubuntu
charly shell ubuntu                       # drops into /home/ubuntu as uid 1000
id                                    # uid=1000(ubuntu) gid=1000(ubuntu)
charly -C image/ubuntu image validate     # remote build.yml resolves distro.ubuntu (inherits debian)
```

## Related images

- `/charly-distros:debian` — sibling deb-family base without adopt mode (Debian 13 ships no pre-existing uid-1000 user).
- `/charly-distros:ubuntu-builder` — multi-stage builder.
- `/charly-coder:ubuntu-coder` — kitchen-sink dev image.
- `/charly-distros:fedora` — RPM-family counterpart.
- `/charly-distros:arch` — pacman-family counterpart.

## Related commands

- `/charly-build:build` — `base_user:` declaration format, which lives in `build.yml distro.ubuntu`.
- `/charly-image:image` — `user_policy:` field + reconciliation.
- `/charly-build:generate` — adopt-vs-create writeBootstrap emission.

## Related layers (cross-distro patterns this base enables)

- `/charly-coder:sshd` — `getent passwd 1000`-based sudoers works for both `user` (create) and `ubuntu` (adopt).
- `/charly-coder:language-runtimes` — Microsoft `dotnet-install.sh` (Ubuntu noble doesn't ship dotnet-sdk-9.0 in main; Microsoft's noble apt repo only has 10.0; the dotnet-install.sh `--channel 9.0` is the cross-distro solution).

## When to use this skill

**MUST be invoked** when:

- Building or troubleshooting the `ubuntu` base image.
- Adding any Ubuntu-based image (it will inherit the adopt-mode `ubuntu:ubuntu` identity by default).
- Debugging `${USER}` / `${HOME}` differences between Ubuntu and other deb-based images (ubuntu-coder → `ubuntu:/home/ubuntu`; debian-coder → `user:/home/user`).
- Understanding why ubuntu-coder's `/etc/sudoers.d/charly-user` says `ubuntu ALL=(ALL) NOPASSWD: ALL` rather than `user`.
