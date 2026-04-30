---
name: selkies-desktop-bootc-bootc
description: |
  kind:vm entity pairing with the /ov-selkies:selkies-desktop-bootc container image.
  source.kind: bootc. Thin pointer skill — composition + layer stack authority
  lives in /ov-selkies:selkies-desktop-bootc (the canonical bootc-VM worked example).
  MUST be invoked before editing selkies-desktop-bootc-bootc in vms.yml.
---

# selkies-desktop-bootc-bootc

`kind: vm` entity that pairs with the `/ov-selkies:selkies-desktop-bootc` container image. The doubled `-bootc` suffix reflects that the paired image already ends in `-bootc`.

**Composition authority: `/ov-selkies:selkies-desktop-bootc`.** This is the **canonical worked example** for the full bootc-VM flow in the project — layer stack (9 layers), port remaps (13000/19222/19224/2250), Fedora 43 external base, distro tags, known caveats, verification recipes. Read it end-to-end before touching this VM.

## VmSpec (from vms.yml)

```yaml
vms:
  selkies-desktop-bootc-bootc:
    source:
      kind: bootc
      image: selkies-desktop-bootc
    disk_size: 40 GiB
    ram: 8G
    cpus: 4
    ssh:
      user: root
      port: 2250
```

## Overrides explained

| Setting | Value | Rationale |
|---|---|---|
| Disk size | 40 GiB | Selkies + Chrome + PipeWire + toolchain transitives — more than openclaw-browser-bootc's 20 GiB |
| RAM | 8G | Chrome + compositor + recorder want headroom |
| CPUs | 4 | 8 G/4 cpus rule of thumb for streaming desktop VMs |
| SSH user | `root` | bootc default |
| SSH port | 2250 | Non-default to avoid colliding with `ov-selkies-desktop*` containers that claim host :2222 |

## Build + run

```bash
ov image build selkies-desktop-bootc
ov vm build selkies-desktop-bootc-bootc --type qcow2
ov vm create selkies-desktop-bootc-bootc
ov vm start selkies-desktop-bootc-bootc
ssh -p 2250 root@localhost
# Desktop accessible at https://localhost:13000 (self-signed)
```

## Cross-References

- `/ov-selkies:selkies-desktop-bootc` — **canonical worked example**: full 9-layer stack, port remaps, Fedora 43 external base, known caveats, end-to-end verification recipes
- `/ov-vms:vms` — VmSpec authoring reference, bootc branch authoring recipe
- `/ov-advanced:vm` — VM lifecycle commands + bootc-specific caveats (rootful storage split, loopback device requirements, nested-container transport)
- `/ov-build:migrate` — `ov migrate vm-spec` legacy conversion
- `/ov-selkies:selkies-desktop` — Selkies streaming desktop metalayer
- `/ov-foundation:bootc-base` — sshd + qemu-guest-agent + bootc-config bundle
- `/ov-foundation:tailscale` — mesh VPN (included in composition)
- `/ov-foundation:keepassxc` — password manager (included in composition)
