---
name: debian-debootstrap
description: |
  Bootstrap-from-scratch Debian rootfs via debootstrap inside a privileged
  builder (from: builder:debootstrap, bootstrap_builder_image:
  debian-debootstrap-builder). Retained for offline/air-gapped builds and as a
  worked example of the from: builder:debootstrap pattern. Lives in the
  overthinkos/debian submodule (image/debian).
  MUST be invoked before building or troubleshooting debian-debootstrap.
---

# debian-debootstrap

Bootstrap-from-scratch Debian root filesystem, built via `debootstrap` inside
the privileged `/ov-distros:debian-debootstrap-builder` container
(`from: builder:debootstrap`, `bootstrap_builder_image: debian-debootstrap-builder`).

> **Lives in `overthinkos/debian`** (git submodule at `image/debian`). Build:
> `ov -C image/debian image build debian-debootstrap`.

## When to use it (and when not)

The canonical Debian base (`/ov-distros:debian`) pulls the upstream-published
`debian:13` OCI image from Docker Hub — that is the recommended, faster path
(no privileged build). This debootstrap variant exists for **offline /
air-gapped** builds and as a worked example of the `from: builder:debootstrap` +
`bootstrap_builder_image:` pattern (the deb-family counterpart of
`/ov-distros:arch-pacstrap` / `/ov-distros:cachyos-pacstrap`).

## Image Properties

| Property | Value |
|----------|-------|
| From | builder:debootstrap |
| bootstrap_builder_image | debian-debootstrap-builder |
| Distro | debian:13, debian |
| Build | deb |
| Home repo | overthinkos/debian (`image/debian`) |

The `debian` distro config (debootstrap suite `trixie`, mirror
`http://deb.debian.org/debian`, base packages, bootloader template) lives in the
main repo's `build.yml` and is remote-included by the submodule.

## Cross-References

- `/ov-distros:debian` — the recommended Docker-Hub base
- `/ov-distros:debian-debootstrap-builder` — the privileged builder it uses
- `/ov-vm:debian` — the VM built via the same debootstrap path

## When to Use This Skill

**MUST be invoked** before building or debugging the Debian debootstrap rootfs.
Invoke BEFORE reading source code or launching Explore agents.
