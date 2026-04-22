---
name: openclaw-browser-bootc-bootc
description: |
  kind:vm entity pairing with the /ov-images:openclaw-browser-bootc container image.
  source.kind: bootc. Thin pointer skill — composition + layer stack authority
  lives in /ov-images:openclaw-browser-bootc. This skill documents only VM overrides.
  MUST be invoked before editing openclaw-browser-bootc-bootc in vms.yml.
---

# openclaw-browser-bootc-bootc

`kind: vm` entity that pairs with the `/ov-images:openclaw-browser-bootc` container image. The doubled `-bootc` suffix reflects that the paired image already ends in `-bootc` — the VM entity is distinguished by the `vms:` namespace, not by its name.

**Composition authority: `/ov-images:openclaw-browser-bootc`.** OpenClaw gateway, Chrome, VNC, PipeWire composition, OCI labels all live there. This skill is a pointer.

## VmSpec (from vms.yml)

```yaml
vms:
  openclaw-browser-bootc-bootc:
    source:
      kind: bootc
      image: openclaw-browser-bootc
    disk_size: 20 GiB
    ram: 4G
    cpus: 2
    ssh:
      user: root
      port: 2222
```

## Overrides explained

| Setting | Value | Rationale |
|---|---|---|
| Disk size | 20 GiB | Minimal bootc install + Chrome + VNC; tighter than desktop-class VMs |
| RAM | 4G | Browser + VNC fits in 4 GiB for a kiosk-style workload |
| CPUs | 2 | Conservative; Chrome + compositor don't need more |
| SSH user | `root` | bootc default; no non-root base user created |
| SSH port | 2222 | Standard — only collision risk is with other VMs on 2222 |

Firmware, machine, network default to VmSpec defaults (see `/ov-vms:vms`).

## Build + run

```bash
# Enable openclaw-browser-bootc in image.yml first (disabled by default)
ov image build openclaw-browser-bootc
ov vm build openclaw-browser-bootc-bootc
ov vm create openclaw-browser-bootc-bootc
ov vm start openclaw-browser-bootc-bootc
ssh -p 2222 root@localhost
```

## Cross-References

- `/ov-images:openclaw-browser-bootc` — **composition authority**: layer stack, OpenClaw gateway, Chrome, VNC, OCI labels
- `/ov-vms:vms` — VmSpec authoring reference, bootc branch authoring recipe
- `/ov:vm` — VM lifecycle commands + bootc-specific caveats
- `/ov:migrate` — `ov migrate vm-spec` legacy conversion
- `/ov-layers:openclaw` — OpenClaw gateway layer
