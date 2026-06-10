---
name: debian
description: |
  Debian bootstrap VM (kind:vm debian-debootstrap) — source.kind: bootstrap via
  debian-debootstrap-builder + debootstrap, ext4 rootfs, uefi-insecure. Plus the
  disposable eval-debian-debootstrap-vm kind:eval bed. Lives in the overthinkos/debian submodule.
  MUST be invoked before editing debian-debootstrap or its eval bed.
---

# debian (VM)

`source.kind: bootstrap` VM that builds a Debian rootfs from scratch via
`debootstrap` (using `/charly-distros:debian-debootstrap-builder`), then boots it
under libvirt/QEMU.

The `debian-debootstrap` VM entity and its `eval-debian-debootstrap-vm` disposable
test bed live in the **`overthinkos/debian`** repo (git submodule at
**`box/debian`**), in that repo's config (its `charly.yml` + per-kind sibling files). The bed is a
`kind: eval` entity (the 2026-05 deploy→eval unification moved repo-shipped
disposable beds out of `charly.yml`), driven by `charly eval run
eval-debian-debootstrap-vm`. Drive the VM lifecycle from the submodule:
`charly -C box/debian vm build debian-debootstrap` +
`charly -C box/debian vm create debian-debootstrap` (or
`charly --repo overthinkos/debian …`).

## VM Configuration (from box/debian/vm.yml)

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
template) comes from the main repo's `build.yml`, flat-imported by the
submodule (a bare-string `import:` item).

## Eval bed

`eval-debian-debootstrap-vm` is a `kind: eval` bed (`target: vm`,
`vm: debian-debootstrap`) that carries `disposable: true`, so
`charly -C box/debian eval run eval-debian-debootstrap-vm` runs the full R10 sequence
unattended (the equivalent `charly update eval-debian-debootstrap-vm` rebuild also works,
since the eval bed is folded into the Deploy map).

## debootstrap path

`charly vm build debian-debootstrap` runs `debootstrap` inside the privileged
`debian-debootstrap-builder` to build the rootfs, then writes a bootable disk +
cloud-init seed ISO. The bootstrap path is privileged + network-heavy
(`debootstrap` downloads the base packages from the Debian mirror). The
Docker-Hub `/charly-distros:debian` container base remains the faster path when you
don't need a VM disk.

## Cross-References

- `/charly-distros:debian-debootstrap-builder` — the builder image this VM uses
- `/charly-distros:debian-debootstrap` — the container equivalent of this bootstrap path
- `/charly-vm:ubuntu` — the Ubuntu sibling bootstrap VM
- `/charly-vm:arch` — the canonical cloud_image VM (BIOS/virtio-gpu/sizing rationale)
- `/charly-vm:vm` — VM lifecycle commands + BIOS/UEFI matrix
- `/charly-vm:vms-catalog` — VmSpec authoring reference

## When to Use This Skill

**MUST be invoked** when editing `debian-debootstrap` or its eval bed, or
authoring a Debian VM. Invoke BEFORE reading source code or launching Explore agents.
