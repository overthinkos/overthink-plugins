---
name: bazzite-bootc
description: |
  kind:vm entity pairing with the /ov-distros:bazzite bootc container image.
  source.kind: bootc. Thin pointer skill — composition + layer stack authority
  lives in /ov-distros:bazzite. This skill documents only the VM-specific fields.
  MUST be invoked before editing bazzite-bootc in image/bootc/overthink.yml.
---

# bazzite-bootc

`kind: vm` entity that pairs with the `/ov-distros:bazzite` container image. `ov vm build bazzite-bootc` runs `bootc install to-disk` against the bazzite image to produce a bootable qcow2/raw disk.

**Composition authority: `/ov-distros:bazzite`.** Layer stack, NVIDIA/CUDA wiring, Kubernetes/Docker tools, desktop apps, OCI labels all live there. This skill is a pointer.

## VmSpec (from image/bootc/overthink.yml)

```yaml
vm:
  bazzite-bootc:
    source:
      kind: bootc
      box: bazzite
    # disk_size, ram, cpus inherit from VmSpec defaults — override locally if the
    # image's workload demands something heavier than 4 GiB / 2 cpus / 20 GiB.
```

Bazzite's full image definition carries a NVIDIA + CUDA + Kubernetes + Docker stack; for realistic workloads author local overrides (`disk_size: 80G`, `ram: 16G`, `cpus: 6`) before `ov vm build`.

## Build + run

```bash
# Built from the bootc submodule.
ov -C image/bootc image build bazzite
ov -C image/bootc vm build bazzite-bootc --transport containers-storage
ov -C image/bootc vm create bazzite-bootc --ram 16G --cpus 6
ov -C image/bootc vm start bazzite-bootc
```

See `/ov-vm:vm` "Known bootc-VM caveats" for the privileged-container `-v /dev:/dev` loopback requirement and glibc-skew preflight.

## Cross-References

- `/ov-distros:bazzite` — **composition authority**: layer stack, NVIDIA/CUDA wiring, OCI labels
- `/ov-vm:vms-catalog` — VmSpec authoring reference, bootc branch authoring recipe
- `/ov-vm:vm` — VM lifecycle commands + bootc-specific caveats
- `/ov-build:migrate` — `ov migrate` legacy conversion
