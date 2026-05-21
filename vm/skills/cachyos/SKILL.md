---
name: cachyos
description: |
  CachyOS bootstrap VM (kind:vm cachyos-vm) — source.kind: bootstrap via
  cachyos-pacstrap-builder + pacstrap, btrfs rootfs, uefi-insecure. Plus the
  disposable cachyos-vm-deploy bed. Lives in the overthinkos/cachyos submodule.
  MUST be invoked before editing cachyos-vm or its deploy bed.
---

# cachyos (VM)

`source.kind: bootstrap` VM that builds a CachyOS rootfs from scratch via
`pacstrap` (using `/ov-distros:cachyos-pacstrap-builder`), then boots it under
libvirt/QEMU.

> **Relocated (2026-05):** the `cachyos-vm` entity and its `cachyos-vm-deploy`
> bed live in the **`overthinkos/cachyos`** repo (git submodule at
> **`image/cachyos`**), in that repo's `vm.yml` / `deploy.yml`. Drive them from
> the submodule: `ov -C image/cachyos vm build cachyos-vm` +
> `ov -C image/cachyos vm create cachyos-vm` (or `ov --repo overthinkos/cachyos …`).

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
the main repo's `build.yml`, remote-included by the submodule.

## Deploy bed

`cachyos-vm-deploy` (`target: vm`, `vm: cachyos-vm`) carries `disposable: true`,
so `ov -C image/cachyos update cachyos-vm-deploy` rebuilds it unattended.

## Known limitation (pacstrap not usable)

The pacstrap bootstrap path shares `/ov-distros:cachyos-pacstrap`'s two
pre-existing failure modes (both observed during R10): an intermittent GPGME
keyring/DB-sync error (`GPGME error: No data` — this is the one `ov vm build
cachyos-vm` hit) and, when keyring init succeeds, an `x86_64_v3` architecture
rejection (`package architecture is not valid`). Both originate in the shared
`build.yml` cachyos distro config, **not** introduced by the submodule split —
proven via an empty `git diff main` on `build.yml` + the pacstrap runner
(`ov/vm_bootstrap.go` is byte-identical). The Docker-Hub `/ov-distros:cachyos`
base is the reliable container path; a fully working CachyOS VM (pacstrap
keyring + `Architecture` fix, or a cloud-image source) is a separate enhancement.

## Cross-References

- `/ov-distros:cachyos-pacstrap-builder` — the builder image this VM uses
- `/ov-distros:cachyos-pacstrap` — the container equivalent of this bootstrap path
- `/ov-vm:arch` — the canonical cloud_image VM (BIOS/virtio-gpu/sizing rationale)
- `/ov-vm:vm` — VM lifecycle commands + BIOS/UEFI matrix
- `/ov-vm:vms-catalog` — VmSpec authoring reference

## When to Use This Skill

**MUST be invoked** when editing `cachyos-vm` or its deploy bed, or authoring a
CachyOS VM. Invoke BEFORE reading source code or launching Explore agents.
