---
name: debian-builder
description: |
  Minimal Debian 13 builder image (pixi + Node.js + build-toolchain) used
  as the multi-stage builder for every image based on Debian — currently
  debian-coder. Produces the pre-compiled pixi envs, npm globals, and
  cargo crates that land in the final runtime image via COPY --from.
  MUST be invoked before building, deploying, configuring, or troubleshooting
  the debian-builder image.
---

# debian-builder

Debian 13 counterpart of `/ov-distros:fedora-builder`. Provides the pixi / npm / cargo build environments so downstream deb-based images (currently `/ov-coder:debian-coder`) get pre-compiled artifacts from a dedicated builder stage without bloating the final image.

## Image Properties

| Property | Value |
|----------|-------|
| Base | `debian` (which = `debian:13` + our bootstrap) |
| Layers | `pixi`, `nodejs`, `build-toolchain` |
| Platforms | `linux/amd64` |
| Registry | `ghcr.io/overthinkos` |
| User | `user` / uid 1000 (create mode — `debian:13` ships no pre-existing uid-1000 account) |

## Full layer stack

1. `/ov-distros:debian` — Debian 13 + our `apt-get update && apt-get install -y --no-install-recommends curl ca-certificates gnupg` bootstrap + go-task binary + `user:user` uid 1000.
2. `/ov-languages:pixi` — pixi package manager + env paths.
3. `/ov-coder:nodejs` — Node.js + npm (generic `nodejs`, not `nodejs24` — Debian packages a current-enough version).
4. `/ov-coder:build-toolchain` — gcc, g++, cmake, autoconf, ninja, pkg-config, and the full set of `-dev` libraries (Debian equivalents of Fedora's `-devel`). Used by cargo crates that link system libs (libva, x264, ffmpeg, wayland, xkbcommon, etc.).

## Role in the build system

Declares `builds: [pixi, npm, cargo]` and is listed as `debian-coder`'s builder for all three types via:

```yaml
debian:
  builder:
    pixi: debian-builder
    npm: debian-builder
    cargo: debian-builder
```

During `ov image build debian-coder`, any layer that ships `pixi.toml` / `package.json` / `Cargo.toml` gets a multi-stage `FROM debian-builder AS <layer>-<builder>-build` section emitted by the generator, then `COPY --from=<stage> --chown=${UID}:${GID}` into the final image. See `/ov-internals:generate-source` for the template.

No AUR equivalent (unlike `/ov-distros:arch-builder`) — AUR is an Arch-only concept.

## Cross-distro sibling builders

- `/ov-distros:fedora-builder` — Fedora 43 equivalent (+ `rpmfusion` for x264/ffmpeg/libva headers).
- `/ov-distros:arch-builder` — Arch Linux equivalent + `yay` for AUR.
- `/ov-distros:ubuntu-builder` — Ubuntu 24.04 equivalent; differs from this image mainly in `user:ubuntu` vs `user:user` (adopt mode — see `/ov-distros:ubuntu`).

## Quick start

```bash
ov image build debian-builder
ov shell debian-builder
```

Typically not invoked directly — it's a build-time dependency of `/ov-coder:debian-coder`.

## Verification

- `ov image list | grep debian-builder` — image present.
- `ov shell debian-builder -- pixi --version && node --version && gcc --version`.

## Related images

- `/ov-distros:debian` — parent base, declares the bootstrap packages in `build.yml`.
- `/ov-coder:debian-coder` — the consumer that this builder exists to serve.
- `/ov-distros:fedora-builder` — RPM-family sibling.
- `/ov-distros:arch-builder` — pacman + AUR sibling.
- `/ov-distros:ubuntu-builder` — Ubuntu 24.04 sibling.

## Related layers

- `/ov-languages:pixi`, `/ov-coder:nodejs`, `/ov-coder:build-toolchain`

## Related commands

- `/ov-build:build` — multi-stage builders, `base_user:` declaration
- `/ov-image:image` — `produce:` + `builder:` fields
- `/ov-build:generate` — COPY-from stage emission

## When to use this skill

**MUST be invoked** when:

- Building or troubleshooting `debian-builder` itself.
- Debugging multi-stage `COPY --from` errors in a `debian-coder` build (the source stage is this image).
- Adding a new builder capability (e.g. a `dotnet` build type) to deb-family images.
