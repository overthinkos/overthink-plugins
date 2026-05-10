---
name: bazzite-ai-bootc
description: |
  kind:vm entity pairing with the /ov-distros:bazzite-ai bootc container image.
  source.kind: bootc. Thin pointer skill — composition + layer stack authority
  lives in /ov-distros:bazzite-ai. This skill documents only the VM-specific fields.
  MUST be invoked before editing bazzite-ai-bootc in vms.yml.
---

# bazzite-ai-bootc

`kind: vm` entity that pairs with the `/ov-distros:bazzite-ai` container image. `ov vm build bazzite-ai-bootc` runs `bootc install to-disk` against the bazzite-ai image to produce a bootable qcow2/raw disk.

**Composition authority: `/ov-distros:bazzite-ai`.** Layer stack, NVIDIA/CUDA wiring, Kubernetes/Docker tools, desktop apps, OCI labels all live there. This skill is a pointer.

## VmSpec (from vms.yml)

```yaml
vms:
  bazzite-ai-bootc:
    source:
      kind: bootc
      image: bazzite-ai
    # disk_size, ram, cpus inherit from VmSpec defaults — override locally if the
    # image's workload demands something heavier than 4 GiB / 2 cpus / 20 GiB.
```

Bazzite-AI's full image definition carries a NVIDIA + CUDA + Kubernetes + Docker stack; for realistic workloads author local overrides (`disk_size: 80G`, `ram: 16G`, `cpus: 6`) before `ov vm build`.

## Build + run

```bash
# Enable bazzite-ai in image.yml first (disabled by default)
ov image build bazzite-ai
ov vm build bazzite-ai-bootc
ov vm create bazzite-ai-bootc --ram 16G --cpus 6
ov vm start bazzite-ai-bootc
```

See `/ov-vm:vm` "Known bootc-VM caveats" for the privileged-container `-v /dev:/dev` loopback requirement and glibc-skew preflight.

## Cross-References

- `/ov-distros:bazzite-ai` — **composition authority**: layer stack, NVIDIA/CUDA wiring, OCI labels
- `/ov-vm:vms-catalog` — VmSpec authoring reference, bootc branch authoring recipe
- `/ov-vm:vm` — VM lifecycle commands + bootc-specific caveats
- `/ov-build:migrate` — `ov migrate vm-spec` legacy conversion
