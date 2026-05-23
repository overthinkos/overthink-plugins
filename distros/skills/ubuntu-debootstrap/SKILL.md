---
name: ubuntu-debootstrap
description: |
  Bootstrap-from-scratch Ubuntu 24.04 rootfs via debootstrap inside a privileged
  builder (from: builder:debootstrap, bootstrap_builder_image:
  ubuntu-debootstrap-builder). Retained for offline/air-gapped builds. Lives in
  the overthinkos/ubuntu submodule (image/ubuntu).
  MUST be invoked before building or troubleshooting ubuntu-debootstrap.
---

# ubuntu-debootstrap

Bootstrap-from-scratch Ubuntu 24.04 (noble) root filesystem, built via
`debootstrap` inside the privileged `/ov-distros:ubuntu-debootstrap-builder`
container (`from: builder:debootstrap`,
`bootstrap_builder_image: ubuntu-debootstrap-builder`).

> **Lives in `overthinkos/ubuntu`** (git submodule at `image/ubuntu`). Build:
> `ov -C image/ubuntu image build ubuntu-debootstrap`.

## When to use it (and when not)

The canonical Ubuntu base (`/ov-distros:ubuntu`) pulls the upstream-published
`ubuntu:24.04` OCI image from Docker Hub — that is the recommended, faster path.
This debootstrap variant exists for **offline / air-gapped** builds and as a
worked example of the `from: builder:debootstrap` pattern.

## Image Properties

| Property | Value |
|----------|-------|
| From | builder:debootstrap |
| bootstrap_builder_image | ubuntu-debootstrap-builder |
| Distro | ubuntu:24.04, ubuntu, debian |
| Build | deb |
| Home repo | overthinkos/ubuntu (`image/ubuntu`) |

The `ubuntu` distro config (`inherits: debian`; debootstrap suite `noble`,
mirror `http://archive.ubuntu.com/ubuntu`, components `main,universe`) lives in
the main repo's `build.yml` and is flat-imported by the submodule (a bare-string
`import:` item). The single imported `build.yml` carries BOTH the `ubuntu` and
`debian` distro configs, so the `inherits: debian` resolution needs no reference
to `overthinkos/debian`.

## Cross-References

- `/ov-distros:ubuntu` — the recommended Docker-Hub base
- `/ov-distros:ubuntu-debootstrap-builder` — the privileged builder it uses
- `/ov-vm:ubuntu` — the VM built via the same debootstrap path

## When to Use This Skill

**MUST be invoked** before building or debugging the Ubuntu debootstrap rootfs.
Invoke BEFORE reading source code or launching Explore agents.
