---
name: cachyos-pacstrap-builder
description: |
  Privileged pacstrap builder image for bootstrapping a CachyOS rootfs from
  scratch. base: arch + the pacstrap-builder layer. Lives in the
  overthinkos/cachyos submodule (image/cachyos).
  MUST be invoked before building or troubleshooting cachyos-pacstrap / cachyos-vm.
---

# cachyos-pacstrap-builder

Privileged builder image used to bootstrap a CachyOS root filesystem via
`pacstrap` inside a container. It is the `bootstrap_builder_image:` for
`/ov-distros:cachyos-pacstrap` and the `builder_image:` for the
`/ov-vm:cachyos` VM.

> **Lives in `overthinkos/cachyos`** (git submodule at `image/cachyos`). It is
> `base: arch` — the `arch` base resolves from the main repo's `arch-base.yml`
> (remote-included in the submodule's `overthink.yml`). Its single layer,
> `pacstrap-builder`, stays in the main repo (shared with
> `/ov-distros:arch-builder`'s pacstrap path) and is pulled by git reference.

## Image Properties

| Property | Value |
|----------|-------|
| Base | arch (from `arch-base.yml`, remote-included) |
| Layer | pacstrap-builder (`@github.com/overthinkos/overthink/layers/pacstrap-builder:<tag>`) |
| Distro | arch |
| Build | pac |
| Home repo | overthinkos/cachyos (`image/cachyos`) |

## Quick Start

```bash
ov -C image/cachyos image build cachyos-pacstrap-builder
```

This is the R10 canary for the submodule's composition machinery: a successful
build proves the remote `arch-base.yml` resolves the `arch` base, the git-ref'd
`pacstrap-builder` layer materializes, and the remote `build.yml` (pacstrap
builder definition + cachyos distro config) is reachable.

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
