---
name: cachyos-pacstrap-builder
description: |
  Privileged pacstrap builder image for bootstrapping a CachyOS rootfs from
  scratch. base: ov.arch (via the `ov` import namespace) + the pacstrap-builder
  layer. Lives in the overthinkos/cachyos submodule (image/cachyos).
  MUST be invoked before building or troubleshooting cachyos-pacstrap / cachyos-vm.
---

# cachyos-pacstrap-builder

Privileged builder image used to bootstrap a CachyOS root filesystem via
`pacstrap` inside a container. It is the `bootstrap_builder_image:` for
`/ov-distros:cachyos-pacstrap` and the `builder_image:` for the
`/ov-vm:cachyos` VM.

> **Lives in `overthinkos/cachyos`** (git submodule at `image/cachyos`). It is
> `base: ov.arch` — the `arch` base from main's `base.yml`, reached because the
> submodule's single `overthink.yml` imports the main repo under the `ov`
> namespace. Its single layer, `pacstrap-builder`, stays in the main repo
> (shared with `/ov-distros:arch-builder`'s pacstrap path) and is pulled by git
> reference.

## Image Properties

| Property | Value |
|----------|-------|
| Base | `ov.arch` (from main's `base.yml`, via the `ov` import namespace) |
| Layer | pacstrap-builder (`@github.com/overthinkos/overthink/layers/pacstrap-builder:<tag>`) |
| Distro | arch |
| Build | pac |
| Home repo | overthinkos/cachyos (`image/cachyos`) |

## Quick Start

```bash
ov -C image/cachyos image build cachyos-pacstrap-builder
```

This is the R10 canary for the submodule's composition machinery: a successful
build proves the `ov` import namespace resolves the `arch` base from main's
`base.yml`, the git-ref'd `pacstrap-builder` layer materializes, and the
flat-imported `build.yml` (pacstrap builder definition + cachyos distro config)
is reachable.

`ov eval image cachyos-pacstrap-builder` runs the build-scope eval: 4 probes pass
(`/usr/sbin/pacstrap`, `arch-install-scripts` installed, `/usr/sbin/grub-install`,
`/usr/sbin/parted`) — i.e. the pacstrap toolchain the builder must ship.

## Cross-References

- `/ov-distros:cachyos` — the Docker-Hub CachyOS base (sibling, no pacstrap)
- `/ov-distros:cachyos-pacstrap` — the rootfs this builder bootstraps
- `/ov-vm:cachyos` — the VM that uses this as `builder_image:`
- `/ov-distros:arch` / `/ov-distros:arch-builder` — the Arch base + builder it mirrors

## When to Use This Skill

**MUST be invoked** when building or debugging the CachyOS pacstrap path.
Invoke BEFORE reading source code or launching Explore agents.
