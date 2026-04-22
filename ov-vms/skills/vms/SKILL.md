---
name: vms
description: |
  Authoring reference for kind:vm entities in vms.yml. Parallel to /ov:layer and /ov:image.
  Covers the VmSpec schema, source.kind discriminator (cloud_image vs bootc),
  base_user adopt pattern, and step-by-step recipes for both source kinds.
  MUST be invoked before authoring or editing vms.yml entries.
---

# vms

`vms.yml` is the authoring surface for `kind: vm` entities — VM primitives that pair with either a remote cloud-image URL (`source.kind: cloud_image`) or an in-repo bootc container image (`source.kind: bootc`). Loaded through `overthink.yml includes:` alongside `image.yml`. Entries are resolved by `LoadUnified` into `VmSpec` Go types (`ov/vm_spec.go`) and consumed by `ov vm build`, `ov vm create`, and `ov deploy add vm:<name>`.

The VM surface parallels the `kind: image` surface: one YAML entry per entity, kind-keyed, discovered through includes. The Go types that back it live in `/ov-dev:vm-spec`; the rendering paths in `/ov-dev:libvirt-renderer` and `/ov-dev:cloud-init-renderer`.

## File layout

```yaml
# vms.yml
vms:
  <name>:
    source:
      kind: cloud_image | bootc
      # cloud_image branch:
      url: https://…
      checksum: { type: sha256, value?: <hex> }
      base_user: arch                     # adopt this account (see Adopt pattern below)
      cache: ~/.cache/ov/vm-images/       # optional override
      # bootc branch:
      image: <kind:image entry name>
      transport: registry | containers-storage | oci | oci-archive
      rootfs: ext4 | xfs | btrfs
      root_size: 10G                      # optional cap; rest of disk stays unpartitioned
      kernel_args: "…"
    # Hardware (both branches):
    disk_size: 20G | "10 GiB"
    ram: 4G | "8192M"
    cpus: 4
    machine: q35 | virt | i440fx           # default: host-native
    firmware: bios | uefi-insecure | uefi-secure  # default: bios
    # Network:
    network:
      mode: user | bridge | nat
      bridge: br0                          # only when mode=bridge
      mac: "52:54:…"                       # optional pin; default stable-from-name
      port_forwards: ["8080:80", …]        # additive to SSH forward
    # SSH + key injection:
    ssh:
      user: arch | root | …                # defaults: cloud_image→"ov", bootc→"root"
      port: 2222                           # host port → guest :22
      key_source: auto | generate | none | /abs/path.pub
      key_injection:
        smbios: auto | enabled | disabled
        cloud_init: auto | enabled | disabled
    # Cloud-init (structured intent):
    cloud_init:
      timezone: UTC
      packages: [sudo, spice-vdagent, …]
      runcmd: ["…", …]
      ov_install:
        strategy: auto | none
    # Libvirt XML knobs (see /ov-dev:libvirt-renderer):
    libvirt:
      devices:
        channels: […]
        graphics: […]
        video: [{model: virtio, vram: 65536, heads: 1, accel3d: false}]
        rng: [{model: virtio, backend: /dev/urandom}]
        memballoon: {model: virtio}
```

## source.kind: cloud_image

Use when the VM is built from an **externally published qcow2** (Arch cloud image from pkgbuild.com, Fedora Cloud, Ubuntu Cloud, Debian Cloud, CentOS Cloud, etc.). The build pipeline fetches the URL, integrity-checks it via sha256 (sidecar auto-resolved when `checksum.value` is empty), creates a qcow2 overlay at `spec.disk_size`, renders a NoCloud seed ISO, and hands off to libvirt/QEMU.

Canonical example: `/ov-vms:arch-cloud-base`. Only existing cloud_image VM in the repo — **read it before authoring another one**. It documents the non-obvious decisions learned the hard way:

- **BIOS firmware is usually the right default.** Distribution cloud images ship both a BIOS boot partition and an EFI System Partition, but the ESP's bootloader binary often has an **embedded grub.cfg that predates the maintainer's latest `/etc/default/grub`** (Arch's upstream issue with `fbcon=nodefer`). BIOS boot reads `/boot/grub/grub.cfg` directly from the root fs, which is always current.
- **virtio-gpu, not QXL.** See `/ov-dev:libvirt-renderer` "video model choice" — virtio-gpu is the modern default for Linux guests.
- **Generous resource sizing.** `pacman -S spice-vdagent` pulls in GTK3 + X11 (~200 MB download, ~1 GB installed); running at 2 GiB RAM stalls cloud-init. Size for the workload: 8 GiB / 4 cpus is reasonable for a workstation-class dev VM.

### Authoring a new cloud_image VM

1. Find the upstream qcow2 URL + verify a sha256 sidecar exists (<url>.SHA256 / .sha256 / .sha256sum).
2. Identify the **pre-existing user account** in the upstream image (`arch`, `ubuntu`, `fedora`, `debian`, `cloud-user`, etc.). This becomes `source.base_user:` — triggers the adopt pattern described below.
3. Start from `/ov-vms:arch-cloud-base` as a template. Change `url`, `base_user`, distro-specific cloud_init `packages:` and `runcmd:`.
4. Pick firmware: default to `bios` unless the upstream image explicitly requires UEFI (e.g., secure boot lock-in).
5. Run `ov vm build <name>` — observe the fetched qcow2 sha256 + rendered seed ISO path.
6. Run `ov vm create <name>` + `ov vm ssh <name>` to verify cloud-init completed.

## source.kind: bootc

Use when the VM is built from an **in-repo bootc container image** (a `kind: image` entry with `bootc: true`). `ov vm build` runs `bootc install to-disk --via-loopback` inside a privileged container to produce the qcow2/raw disk.

The 4 bootc VMs currently shipped (each with a dedicated thin skill):

| VM entity | Paired container image | Skill |
|---|---|---|
| `aurora-bootc` | `/ov-images:aurora` | `/ov-vms:aurora-bootc` |
| `bazzite-ai-bootc` | `/ov-images:bazzite-ai` | `/ov-vms:bazzite-ai-bootc` |
| `openclaw-browser-bootc-bootc` | `/ov-images:openclaw-browser-bootc` | `/ov-vms:openclaw-browser-bootc-bootc` |
| `selkies-desktop-bootc-bootc` | `/ov-images:selkies-desktop-bootc` | `/ov-vms:selkies-desktop-bootc-bootc` |

The `-bootc` suffix is doubled on some entries because the paired container image already ends in `-bootc` (the VM is distinguished from an equivalent container-form deploy by the `vms:` namespacing, not by the name).

### Authoring a new bootc VM

1. Ensure the paired container image has `bootc: true` declared and builds cleanly.
2. Add a `vms:` entry with `source.kind: bootc` + `source.image: <entry-name>`.
3. Size disk/ram/cpus for the workload (see `/ov-images:<name>` VM Configuration section for the authoritative numbers).
4. Run `ov vm build <vm-name>`. See `/ov:vm` known-caveats section for bootc-specific gotchas (rootful storage split, nested-container `--transport containers-storage`, loopback device mount namespace).

## Adopt pattern: base_user (cloud_image only)

Mirrors the container-side `base_user:` + `user_policy: adopt` pattern documented in `/ov:image` "user_policy". The key insight: **don't recreate accounts cloud-init already shipped** — just append the SSH pubkey and move on.

When `source.base_user:` is set, the cloud-init renderer (`/ov-dev:cloud-init-renderer::composeUsers`) emits a merge-by-name entry:

```yaml
# Rendered user-data
users:
  - default
  - name: <base_user>
    ssh_authorized_keys:
      - ssh-ed25519 AAAA…
```

cloud-init interprets `users: [default, {name: <base_user>}]` as "keep the distro's default account untouched, append SSH key to the named account". Result: no `useradd`, no sudoers rewrite, no shell change, no home-directory relocation — just the pubkey lands in `~<user>/.ssh/authorized_keys` on first boot.

`spec.ssh.user` defaults to `source.base_user`, so `ov vm ssh <name>` connects as the adopted account without extra declaration.

Leave `base_user:` empty **only** when the upstream has no default account — in which case author a full custom user entry in `cloud_init.users:` with sudo/groups/shell fields. Don't do this when adopting works; `useradd`-at-first-boot races with other cloud-init modules and is harder to reason about.

## ov_install.strategy: auto (cloud_image)

`ov_install.strategy: auto` in `cloud_init:` wires `ov`'s in-guest installer (`/ov-dev:cloud-init-renderer` → emitted `runcmd:` entries) so the provisioned VM comes up with `ov` already installed. Lets `ov deploy add vm:<name>` apply host-deploy-style layer recipes inside the VM over SSH without a bootstrap round-trip. See `/ov-dev:cloud-init-renderer` for the emission + handshake.

`strategy: none` skips the step entirely — useful when the VM will be managed by something other than `ov` after provisioning.

## Validation rules

Load-time errors raised by `ValidateVmSpec` (`ov/libvirt_validate.go`, see `/ov-dev:vm-spec`):

- `source.kind` must be one of `cloud_image`, `bootc`.
- `cloud_image` branch requires `url:` populated.
- `bootc` branch requires `image:` populated and pointing at a resolvable `kind:image` entry.
- `firmware:` must be one of `bios`, `uefi-insecure`, `uefi-secure`.
- `network.mode:` must be one of `user`, `bridge`, `nat`.
- `ssh.key_source:` must parse as `auto`, `generate`, `none`, or an absolute path.
- `ssh.key_injection.smbios` / `.cloud_init` must be one of `auto`, `enabled`, `disabled`.
- `libvirt:` structure is schema-validated via `ValidateLibvirtConfig` — invalid snippets fail fast.

## Migration from legacy (image.bootc / image.vm / image.libvirt)

Projects predating this schema had three coupled fields on `kind: image` entries: `bootc: true`, `vm: {...}`, `libvirt: [...]`. All three were deleted in the hard cutover. Conversion is one-shot:

```bash
ov migrate vm-spec
```

Idempotent. Harvests the legacy fields into `vms:` entries, preserving any pre-existing `vms:` keys. See `/ov:migrate` for the full command reference and `/ov-dev:cutover-policy` for why hard-cutover was the chosen policy.

## Cross-References

- `/ov:vm` — the `ov vm build/create/start/stop/ssh/console` command family
- `/ov:migrate` — `ov migrate vm-spec` conversion from legacy
- `/ov:deploy` — `ov deploy add vm:<name>` for in-guest layer application
- `/ov-vms:arch-cloud-base` — canonical cloud_image VM
- `/ov-vms:aurora-bootc`, `/ov-vms:bazzite-ai-bootc`, `/ov-vms:openclaw-browser-bootc-bootc`, `/ov-vms:selkies-desktop-bootc-bootc` — bootc VMs
- `/ov-dev:vm-spec` — Go type reference
- `/ov-dev:libvirt-renderer` — libvirt XML emission
- `/ov-dev:cloud-init-renderer` — NoCloud seed ISO + user-data emission
- `/ov-dev:ovmf` — UEFI firmware path resolution (when `firmware:` ≠ `bios`)
- `/ov-dev:vm-deploy-target` — `VmDeployTarget` in the InstallPlan pipeline
- `/ov-dev:cutover-policy` — Hard Cutover by Default policy
- `/ov-layers:cloud-init` — guest-side cloud-init layer (pairs with host-side `cloud_init:` emission)
- `/ov-layers:qemu-guest-agent` — virtio-serial channel for host↔guest comms
