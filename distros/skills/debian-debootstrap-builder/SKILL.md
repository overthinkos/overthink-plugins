---
name: debian-debootstrap-builder
description: |
  Privileged debootstrap builder image (base: debian:13) — ships the
  debootstrap toolchain used to bootstrap a Debian rootfs from scratch for
  debian-debootstrap and the debian-debootstrap VM. Lives in the
  overthinkos/debian submodule (image/debian).
  MUST be invoked before building or troubleshooting debian-debootstrap-builder.
---

# debian-debootstrap-builder

Privileged builder image that ships the **debootstrap** toolchain (the
`debootstrap-builder` layer). It is the `from: builder:debootstrap` /
`bootstrap_builder_image:` target for `/charly-distros:debian-debootstrap` and the
`/charly-vm:debian` bootstrap VM.

> **Lives in `overthinkos/debian`** (git submodule at `image/debian`). Build:
> `charly -C image/debian image build debian-debootstrap-builder`.

## Image Properties

| Property | Value |
|----------|-------|
| Base | `debian:13` |
| Distro | debian:13, debian |
| Build | deb |
| Layer | `debootstrap-builder` (pulled by github reference from the main repo) |
| Home repo | overthinkos/debian (`image/debian`) |

The `debootstrap-builder` layer (debootstrap + arch-install-scripts equivalents,
grub, parted) **stays in the main repo** and is shared with
`/charly-distros:ubuntu-debootstrap-builder` — both pull it by github reference. The
`debian` distro config (debootstrap suite/mirror/base packages, bootloader
template) lives in the main repo's `build.yml` and is flat-imported by the
submodule (a bare-string `import:` item).

## Cross-References

- `/charly-distros:debian` — the Docker-Hub-equivalent base (`base: debian:13`)
- `/charly-distros:debian-debootstrap` — the bootstrap-from-scratch rootfs it builds
- `/charly-vm:debian` — the VM built via the same debootstrap path
- `/charly-distros:ubuntu-debootstrap-builder` — the Ubuntu sibling (also `base: debian:13`)

## When to Use This Skill

**MUST be invoked** before building or debugging the Debian debootstrap builder.
Invoke BEFORE reading source code or launching Explore agents.
