---
name: debian
description: |
  Debian bootstrap VM (kind:vm debian-debootstrap) — source.kind: bootstrap via
  debian-debootstrap-builder + debootstrap, ext4 rootfs, uefi-insecure. Plus the
  disposable debian-debootstrap-vm bed. Lives in the overthinkos/debian submodule.
  MUST be invoked before editing debian-debootstrap or its deploy bed.
---

# debian (VM)

`source.kind: bootstrap` VM that builds a Debian rootfs from scratch via
`debootstrap` (using `/ov-distros:debian-debootstrap-builder`), then boots it
under libvirt/QEMU.

> **Relocated (2026-05):** the `debian-debootstrap` VM entity and its
> `debian-debootstrap-vm` bed live in the **`overthinkos/debian`** repo (git
> submodule at **`image/debian`**), in that repo's `vm.yml` / `deploy.yml`.
> Drive them from the submodule: `ov -C image/debian vm build debian-debootstrap`
> + `ov -C image/debian vm create debian-debootstrap` (or
> `ov --repo overthinkos/debian …`).

## VM Configuration (from image/debian/vm.yml)

| Setting | Value |
|---|---|
| Source | `kind: bootstrap`, `builder: debootstrap`, `builder_image: debian-debootstrap-builder` |
| Distro | debian |
| rootfs | ext4 |
| Disk / RAM / CPU | 20G / 4G / 2 |
| Machine / firmware | q35 / uefi-insecure |
| Network | user mode |
| SSH | user `debian`, port 12227, key_source generate |

The `debian` distro config (debootstrap suite/mirror, base packages, bootloader
template) comes from the main repo's `build.yml`, remote-included by the
submodule.

## Deploy bed

`debian-debootstrap-vm` (`target: vm`, `vm: debian-debootstrap`) carries
`disposable: true`, so `ov -C image/debian update debian-debootstrap-vm`
rebuilds it unattended.

## debootstrap path

`ov vm build debian-debootstrap` runs `debootstrap` inside the privileged
`debian-debootstrap-builder` to build the rootfs, then writes a bootable disk +
cloud-init seed ISO. The bootstrap path is privileged + network-heavy
(`debootstrap` downloads the base packages from the Debian mirror). The
Docker-Hub `/ov-distros:debian` container base remains the faster path when you
don't need a VM disk.

## Cross-References

- `/ov-distros:debian-debootstrap-builder` — the builder image this VM uses
- `/ov-distros:debian-debootstrap` — the container equivalent of this bootstrap path
- `/ov-vm:ubuntu` — the Ubuntu sibling bootstrap VM
- `/ov-vm:arch` — the canonical cloud_image VM (BIOS/virtio-gpu/sizing rationale)
- `/ov-vm:vm` — VM lifecycle commands + BIOS/UEFI matrix
- `/ov-vm:vms-catalog` — VmSpec authoring reference

## When to Use This Skill

**MUST be invoked** when editing `debian-debootstrap` or its deploy bed, or
authoring a Debian VM. Invoke BEFORE reading source code or launching Explore agents.
