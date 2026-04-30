---
name: aurora-bootc
description: |
  kind:vm entity pairing with the /ov-foundation:aurora bootc container image.
  source.kind: bootc. Thin pointer skill — composition + layer stack authority
  lives in /ov-foundation:aurora. This skill documents only the VM resource sizing.
  MUST be invoked before editing aurora-bootc in vms.yml.
---

# aurora-bootc

`kind: vm` entity that pairs with the `/ov-foundation:aurora` container image. `ov vm build aurora-bootc` runs `bootc install to-disk` against the aurora image to produce a bootable qcow2/raw disk.

**Composition authority: `/ov-foundation:aurora`.** Layer stack, base image, tests, and OCI labels all live there. This skill is a pointer; it only documents the VM-specific delta (disk_size / ram / cpus).

## VmSpec (from vms.yml)

```yaml
vms:
  aurora-bootc:
    source:
      kind: bootc
      image: aurora
    disk_size: 80 GiB
    ram: 12G
    cpus: 4
```

## Resource sizing rationale

| Setting | Value | Rationale |
|---|---|---|
| Disk size | 80 GiB | Aurora DX ships full KDE + NVIDIA drivers + toolchain; baseline image footprint is ~30 GiB uncompressed |
| RAM | 12G | KDE desktop + NVIDIA stack + ov toolchain working set |
| CPUs | 4 | Standard workstation-class dev VM allocation |

Firmware, machine, network, and SSH settings fall back to VmSpec defaults (see `/ov-vms:vms`). Override locally in vms.yml if the paired image changes its init system or boot requirements.

## Build + run

```bash
# Enable aurora in image.yml first (disabled by default)
ov image build aurora          # container image must exist before the VM disk install step
ov vm build aurora-bootc
ov vm create aurora-bootc
ov vm start aurora-bootc
ov vm ssh aurora-bootc
```

See `/ov-advanced:vm` "Known bootc-VM caveats" for the rootful storage split (`engine.rootful=sudo` needs `sudo podman load` of the saved tarball to reach root's storage) and the nested-container `--transport containers-storage` pattern.

## Cross-References

- `/ov-foundation:aurora` — **composition authority**: layer stack, base image, OCI labels
- `/ov-vms:vms` — VmSpec authoring reference, bootc branch authoring recipe
- `/ov-advanced:vm` — VM lifecycle commands + bootc-specific caveats
- `/ov-build:migrate` — `ov migrate vm-spec` legacy conversion
- `/ov-foundation:bootc-base` — sshd + qemu-guest-agent + bootc-config bundle
