---
name: cachyos
description: |
  CachyOS bootstrap VM (kind:vm cachyos-vm) — source.kind: bootstrap via
  cachyos-pacstrap-builder + pacstrap, btrfs rootfs, uefi-insecure. Plus the
  disposable cachyos-vm-deploy kind:eval bed. Lives in the overthinkos/cachyos submodule.
  MUST be invoked before editing cachyos-vm or its eval bed.
---

# cachyos (VM)

`source.kind: bootstrap` VM that builds a CachyOS rootfs from scratch via
`pacstrap` (using `/ov-distros:cachyos-pacstrap-builder`), then boots it under
libvirt/QEMU.

The `cachyos-vm` entity and its `cachyos-vm-deploy` disposable test bed live in
the **`overthinkos/cachyos`** repo (git submodule at **`image/cachyos`**),
inlined in that repo's single `overthink.yml`. The bed is a `kind: eval` entity
(the 2026-05 deploy→eval unification moved repo-shipped disposable beds out of
`deploy.yml`), driven by `ov eval run cachyos-vm-deploy`. Drive the VM lifecycle
from the submodule: `ov -C image/cachyos vm build cachyos-vm` +
`ov -C image/cachyos vm create cachyos-vm` (or `ov --repo overthinkos/cachyos …`).

## VM Configuration (from image/cachyos/vm.yml)

| Setting | Value |
|---|---|
| Source | `kind: bootstrap`, `builder: pacstrap`, `builder_image: cachyos-pacstrap-builder` |
| Distro | cachyos |
| rootfs | btrfs |
| Disk / RAM / CPU | 25G / 4G / 2 |
| Machine / firmware | q35 / uefi-insecure |
| Network | user mode |
| SSH | user `cachy`, port 12226, key_source generate |

The `cachyos` distro config (base packages, keyring, mirrors, repos) comes from
the main repo's `build.yml`, flat-imported by the submodule (a bare-string
`import:` item).

## Eval bed

`cachyos-vm-deploy` is a `kind: eval` bed (`target: vm`, `vm: cachyos-vm`) that
carries `disposable: true`, so `ov -C image/cachyos eval run cachyos-vm-deploy`
runs the full R10 sequence unattended (the equivalent `ov update
cachyos-vm-deploy` rebuild also works, since the eval bed is folded into the
Deploy map).

## pacstrap path

`ov vm build cachyos-vm` builds a bootable disk end-to-end. The
`vm_bootstrap.go` path uses the same `renderPacstrapExtraConf` helper as the
image path, which emits the per-repo `SigLevel` (required for GPGME) and an
`[options] Architecture = x86_64 x86_64_v3` directive (so `linux-cachyos`
passes the architecture check).

pacstrap installs `linux-cachyos` (x86_64_v3), mkinitcpio generates the
initramfs (its `autodetect` hook warns in the chroot and falls back to
all-modules — non-fatal), and a bootable `disk.qcow2` (~3.5 GB) + cloud-init
`seed.iso` are written. The Docker-Hub `/ov-distros:cachyos` container base
remains the faster path when you don't need a VM disk.

## Cross-References

- `/ov-distros:cachyos-pacstrap-builder` — the builder image this VM uses
- `/ov-distros:cachyos-pacstrap` — the container equivalent of this bootstrap path
- `/ov-vm:arch` — the canonical cloud_image VM (BIOS/virtio-gpu/sizing rationale)
- `/ov-vm:vm` — VM lifecycle commands + BIOS/UEFI matrix
- `/ov-vm:vms-catalog` — VmSpec authoring reference

## When to Use This Skill

**MUST be invoked** when editing `cachyos-vm` or its eval bed, or authoring a
CachyOS VM. Invoke BEFORE reading source code or launching Explore agents.
