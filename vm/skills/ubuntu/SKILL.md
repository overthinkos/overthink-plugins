---
name: ubuntu
description: |
  Ubuntu bootstrap VM (kind:vm ubuntu-debootstrap) — source.kind: bootstrap via
  ubuntu-debootstrap-builder + debootstrap, ext4 rootfs, uefi-insecure. Plus the
  disposable eval-ubuntu-debootstrap-vm kind:eval bed. Lives in the overthinkos/ubuntu submodule.
  MUST be invoked before editing ubuntu-debootstrap or its eval bed.
---

# ubuntu (VM)

`source.kind: bootstrap` VM that builds an Ubuntu 24.04 rootfs from scratch via
`debootstrap` (using `/charly-distros:ubuntu-debootstrap-builder`), then boots it
under libvirt/QEMU.

The `ubuntu-debootstrap` VM entity and its `eval-ubuntu-debootstrap-vm` disposable
test bed live in the **`overthinkos/ubuntu`** repo (git submodule at
**`box/ubuntu`**), in that repo's config (its `charly.yml` + per-kind sibling files). The bed is a
`kind: eval` entity (the 2026-05 deploy→eval unification moved repo-shipped
disposable beds out of `deploy.yml`), driven by `charly eval run
eval-ubuntu-debootstrap-vm`. Drive the VM lifecycle from the submodule:
`charly -C box/ubuntu vm build ubuntu-debootstrap` +
`charly -C box/ubuntu vm create ubuntu-debootstrap` (or
`charly --repo overthinkos/ubuntu …`).

## VM Configuration (from box/ubuntu/vm.yml)

| Setting | Value |
|---|---|
| Source | `kind: bootstrap`, `builder: debootstrap`, `builder_image: ubuntu-debootstrap-builder` |
| Distro | ubuntu |
| rootfs | ext4 |
| Disk / RAM / CPU | 20G / 4G / 2 |
| Machine / firmware | q35 / uefi-insecure |
| Network | user mode |
| SSH | user `ubuntu`, port 12229, key_source generate |

The `ubuntu` distro config (`inherits: debian`; debootstrap suite `noble`,
mirror, base packages) comes from the main repo's `build.yml`, flat-imported
by the submodule (a bare-string `import:` item). The single imported `build.yml`
carries BOTH distro configs, so `inherits: debian` resolves without referencing
`overthinkos/debian`.

## Eval bed

`eval-ubuntu-debootstrap-vm` is a `kind: eval` bed (`target: vm`,
`vm: ubuntu-debootstrap`) that carries `disposable: true`, so
`charly -C box/ubuntu eval run eval-ubuntu-debootstrap-vm` runs the full R10 sequence
unattended (the equivalent `charly update eval-ubuntu-debootstrap-vm` rebuild also works,
since the eval bed is folded into the Deploy map).

## debootstrap path

`charly vm build ubuntu-debootstrap` runs `debootstrap` inside the privileged
`ubuntu-debootstrap-builder` to build the rootfs, then writes a bootable disk +
cloud-init seed ISO. The bootstrap path is privileged + network-heavy
(`debootstrap` downloads the base packages from the Ubuntu mirror). The
Docker-Hub `/charly-distros:ubuntu` container base remains the faster path when you
don't need a VM disk.

## Cross-References

- `/charly-distros:ubuntu-debootstrap-builder` — the builder image this VM uses
- `/charly-distros:ubuntu-debootstrap` — the container equivalent of this bootstrap path
- `/charly-vm:debian` — the Debian sibling bootstrap VM
- `/charly-vm:arch` — the canonical cloud_image VM (BIOS/virtio-gpu/sizing rationale)
- `/charly-vm:vm` — VM lifecycle commands + BIOS/UEFI matrix
- `/charly-vm:vms-catalog` — VmSpec authoring reference

## When to Use This Skill

**MUST be invoked** when editing `ubuntu-debootstrap` or its eval bed, or
authoring an Ubuntu VM. Invoke BEFORE reading source code or launching Explore agents.
