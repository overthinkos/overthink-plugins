---
name: cachyos-pacstrap-builder
description: |
  Privileged pacstrap builder image for bootstrapping a CachyOS rootfs from
  scratch. base: arch.arch (via the `arch` import namespace) + the pacstrap-builder
  layer. Lives in the overthinkos/cachyos submodule (box/cachyos).
  MUST be invoked before building or troubleshooting cachyos-pacstrap / cachyos-vm.
---

# cachyos-pacstrap-builder

Privileged builder image used to bootstrap a CachyOS root filesystem via
`pacstrap` inside a container. It is the `bootstrap_builder_image:` for
`/charly-distros:cachyos-pacstrap` and the `builder_image:` for the
`/charly-vm:cachyos` VM.

> **Lives in `overthinkos/cachyos`** (git submodule at `box/cachyos`). It is
> `base: arch.arch` — the `arch` base from the **`overthinkos/arch`** submodule,
> reached because the cachyos `charly.yml` imports that submodule under the `arch`
> namespace. Its single layer, `pacstrap-builder`, lives in the main repo
> (shared with `/charly-distros:arch-builder`'s pacstrap path) and is pulled by git
> reference.

## Box Properties

| Property | Value |
|----------|-------|
| Base | `arch.arch` (from the `overthinkos/arch` submodule, via the `arch` import namespace) |
| Layer | pacstrap-builder (`@github.com/overthinkos/overthink/candy/pacstrap-builder:<tag>`) |
| Distro | arch |
| Build | pac |
| Home repo | overthinkos/cachyos (`box/cachyos`) |

## Quick Start

```bash
charly -C box/cachyos box build cachyos-pacstrap-builder
```

This is the R10 canary for the submodule's composition machinery: a successful
build proves the `arch` import namespace resolves the `arch` base from the
`overthinkos/arch` submodule, the git-ref'd `pacstrap-builder` layer
materializes, and the embedded build vocabulary (pacstrap builder definition +
cachyos distro config) is reachable.

`charly eval box cachyos-pacstrap-builder` runs the build-scope eval: 4 probes pass
(`/usr/sbin/pacstrap`, `arch-install-scripts` installed, `/usr/sbin/grub-install`,
`/usr/sbin/parted`) — i.e. the pacstrap toolchain the builder must ship.

## Cross-References

- `/charly-distros:cachyos` — the Docker-Hub CachyOS base (sibling, no pacstrap)
- `/charly-distros:cachyos-pacstrap` — the rootfs this builder bootstraps
- `/charly-vm:cachyos` — the VM that uses this as `builder_image:`
- `/charly-distros:arch` / `/charly-distros:arch-builder` — the Arch base + builder it mirrors

## When to Use This Skill

**MUST be invoked** when building or debugging the CachyOS pacstrap path.
Invoke BEFORE reading source code or launching Explore agents.
