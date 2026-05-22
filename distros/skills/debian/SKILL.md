---
name: debian
description: |
  Base Debian 13 trixie image. Root of the image hierarchy for Debian builds
  that run as uid 1000 `user` (create mode — Debian 13 ships no pre-existing
  uid-1000 account). Owned by the overthinkos/debian submodule (image/debian);
  consumed by no main-repo image.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  any Debian-based image.
---

# debian

Base Debian 13 (trixie) image. Root of the Debian image hierarchy.

The Debian family lives in the **`overthinkos/debian`** repo (git submodule at
**`image/debian`**). The `debian` base is **owned there** (in that repo's
`image.yml`) and composes the main repo's layers + shared `build.yml` (which
keeps the `debian` distro config + the `deb` format + the `debootstrap` builder
template) by git reference. Build it from the submodule:
`ov -C image/debian image build debian` (or `ov --repo overthinkos/debian image build debian`).
Ubuntu — the deb-family sibling — lives in its own **`overthinkos/ubuntu`** repo
(see `/ov-distros:ubuntu`). Nothing in main consumes any Debian image, so there
is **no main ↔ debian coupling**.

## Image Properties

| Property | Value |
|----------|-------|
| Base | `debian:13` |
| Pkg | `deb` |
| Distro tags | `["debian:13", "debian"]` |
| Layers | (none — base image only) |
| Platforms | `linux/amd64` |
| User | `user` / uid 1000 (**create mode**) |
| Home | `/home/user` |
| Registry | `ghcr.io/overthinkos` |

## User model — create, not adopt

`build.yml distro.debian` **does not** declare a `base_user:` block, because the upstream `debian:13` base image ships no pre-existing uid-1000 account. `user_policy: auto` (the default for any downstream image) falls through to create mode — the generator emits an idempotent `useradd -m -u 1000 -g 1000 user` during bootstrap.

This is the intentional asymmetry vs `/ov-distros:ubuntu`, which DOES ship `ubuntu:ubuntu` at uid 1000 and declares `base_user:` to adopt that identity. See `/ov-image:image` "user_policy" and `/ov-build:build` "base_user" for the full decision table.

If you build a downstream image on `debian:13-cloud` (or a similar variant that ships a pre-existing `debian` account), override this by adding a `base_user:` block in your project's `build.yml` override.

## Bootstrap

```dockerfile
FROM debian:13
RUN --mount=type=cache,dst=/var/cache/apt,sharing=locked
    --mount=type=cache,dst=/var/lib/apt,sharing=locked
    apt-get update && apt-get install -y --no-install-recommends curl ca-certificates gnupg && \
    ... install go-task binary ...
RUN if ! getent passwd 1000 >/dev/null 2>&1; then
      (getent group 1000 >/dev/null 2>&1 || groupadd -g 1000 user) &&
      useradd -m -u 1000 -g 1000 -s /bin/bash user;
    fi
WORKDIR /home/user
USER 1000
```

`gnupg` is in the bootstrap package set because downstream layers with `deb.repos[].key` (GitHub CLI, Docker, Kubernetes, Tailscale, Microsoft) call `gpg --dearmor` to convert ASCII-armored keys into `/etc/apt/keyrings/<name>.gpg`. Without `gnupg` the apt-repo stages fail with `gpg: not found`.

## Downstream / sibling entries (all in overthinkos/debian)

- `/ov-distros:debian-builder` — pixi/npm/cargo multi-stage builder on this base.
- `/ov-coder:debian-coder` — kitchen-sink dev image on this base.
- `/ov-distros:debian-debootstrap-builder` — privileged debootstrap builder (`base: debian:13`).
- `/ov-distros:debian-debootstrap` — bootstrap-from-scratch rootfs (`from: builder:debootstrap`).
- `/ov-vm:debian` — the `debian-debootstrap` bootstrap VM + `debian-debootstrap-vm` bed.

## Verification

```bash
ov -C image/debian image build debian
ov shell debian                       # drops into /home/user as uid 1000
id                                    # uid=1000(user) gid=1000(user)
ov -C image/debian image validate     # remote build.yml + layer refs resolve
```

## Related images

- `/ov-distros:ubuntu` — deb-family sibling base in the separate `overthinkos/ubuntu` submodule; ships `base_user:` + adopts ubuntu:ubuntu.
- `/ov-distros:debian-builder` — multi-stage builder on top of this image.
- `/ov-coder:debian-coder` — kitchen-sink dev image on this base.
- `/ov-distros:fedora` — RPM-family counterpart.
- `/ov-distros:arch` — pacman-family counterpart (separate `overthinkos/arch` submodule).

## Related commands

- `/ov-build:build` — `base_user:` declaration format + bootstrap packages.
- `/ov-image:image` — `user_policy:` reconciliation.
- `/ov-build:generate` — writeBootstrap emission.

## When to use this skill

**MUST be invoked** when:

- Building or troubleshooting the `debian` base image.
- Adding any deb-family image that inherits from `debian` (not `ubuntu`).
- Debugging uid-1000 user issues on a Debian-based image — the answer is almost always "create mode fires, user named `user`."
