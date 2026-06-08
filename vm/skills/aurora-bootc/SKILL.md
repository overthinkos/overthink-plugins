---
name: aurora-bootc
description: |
  kind:vm entity pairing with the /charly-distros:aurora bootc container image.
  source.kind: bootc. Thin pointer skill — composition + layer stack authority
  lives in /charly-distros:aurora. This skill documents only the VM resource sizing.
  MUST be invoked before editing aurora-bootc in image/bootc/charly.yml.
---

# aurora-bootc

`kind: vm` entity that pairs with the `/charly-distros:aurora` container image. `charly vm build aurora-bootc` runs `bootc install to-disk` against the aurora image to produce a bootable qcow2/raw disk.

**Composition authority: `/charly-distros:aurora`.** Layer stack, base image, tests, and OCI labels all live there. This skill is a pointer; it only documents the VM-specific delta (disk_size / ram / cpus).

## VmSpec (from image/bootc/charly.yml)

```yaml
vms:
  aurora-bootc:
    source:
      kind: bootc
      box: aurora
    disk_size: 80 GiB
    ram: 12G
    cpus: 4
```

## Resource sizing rationale

| Setting | Value | Rationale |
|---|---|---|
| Disk size | 80 GiB | Aurora DX ships full KDE + NVIDIA drivers + toolchain; baseline image footprint is ~30 GiB uncompressed |
| RAM | 12G | KDE desktop + NVIDIA stack + charly toolchain working set |
| CPUs | 4 | Standard workstation-class dev VM allocation |

Firmware, machine, network, and SSH settings fall back to VmSpec defaults (see `/charly-vm:vms-catalog`). Override locally in vm.yml if the paired image changes its init system or boot requirements.

## Build + run

```bash
# Build the aurora container image first (the VM disk install consumes it)
charly box build aurora          # container image must exist before the VM disk install step
charly vm build aurora-bootc
charly vm create aurora-bootc
charly vm start aurora-bootc
charly vm ssh aurora-bootc
```

See `/charly-vm:vm` "Known bootc-VM caveats" for the rootful storage split (`engine.rootful=sudo` needs `sudo podman load` of the saved tarball to reach root's storage) and the nested-container `--transport containers-storage` pattern.

## Cross-References

- `/charly-distros:aurora` — **composition authority**: layer stack, base image, OCI labels
- `/charly-vm:vms-catalog` — VmSpec authoring reference, bootc branch authoring recipe
- `/charly-vm:vm` — VM lifecycle commands + bootc-specific caveats
- `/charly-build:migrate` — `charly migrate` legacy conversion
- `/charly-distros:bootc-base` — sshd + qemu-guest-agent + bootc-config bundle
