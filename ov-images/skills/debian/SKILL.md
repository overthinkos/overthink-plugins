---
name: debian
description: |
  Base Debian 13 trixie image. Root of the image hierarchy for deb-based
  builds that run as uid 1000 `user` (create mode — Debian 13 ships no
  pre-existing uid-1000 account). Enabled 2026-04 as part of Phase A–F.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  any Debian-based image.
---

# debian

Base Debian 13 (trixie) image. Root of the deb-family image hierarchy alongside `/ov-images:ubuntu`.

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

This is the intentional asymmetry vs `/ov-images:ubuntu`, which DOES ship `ubuntu:ubuntu` at uid 1000 and declares `base_user:` to adopt that identity. See `/ov:image` "user_policy" and `/ov:build` "base_user" for the full decision table.

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

## Downstream images

Currently the single consumer is `/ov-images:debian-builder` (itself the builder for `/ov-images:debian-coder`).

## Verification

```bash
ov image build debian
ov shell debian                       # drops into /home/user as uid 1000
id                                    # uid=1000(user) gid=1000(user)
```

## Related images

- `/ov-images:ubuntu` — sibling deb-family base; ships `base_user:` + adopts ubuntu:ubuntu.
- `/ov-images:debian-builder` — multi-stage builder on top of this image.
- `/ov-images:debian-coder` — kitchen-sink dev image on this base.
- `/ov-images:fedora` — RPM-family counterpart.
- `/ov-images:archlinux` — pacman-family counterpart.

## Related commands

- `/ov:build` — `base_user:` declaration format + bootstrap packages.
- `/ov:image` — `user_policy:` reconciliation.
- `/ov:generate` — writeBootstrap emission.

## When to use this skill

**MUST be invoked** when:

- Building or troubleshooting the `debian` base image.
- Adding any deb-family image that inherits from `debian` (not `ubuntu`).
- Debugging uid-1000 user issues on a Debian-based image — the answer is almost always "create mode fires, user named `user`."
