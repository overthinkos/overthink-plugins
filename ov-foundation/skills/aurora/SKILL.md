---
name: aurora
description: |
  Aurora DX bootc image with NVIDIA, SSH, ov toolchain, and Go.
  Currently disabled. Enable in image.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the aurora image.
---

# aurora

Bootc VM image based on Universal Blue's Aurora DX with NVIDIA drivers, SSH access, full ov toolchain, and Go compiler.

## Image Properties

| Property | Value |
|----------|-------|
| Base | ghcr.io/ublue-os/aurora-dx-nvidia-open:stable-daily-43.20260305 |
| Bootc | true |
| Layers | agent-forwarding, sshd, ov-full, golang |
| Platforms | linux/amd64 |
| Status | **disabled** (set `enabled: true` in image.yml) |
| Registry | ghcr.io/overthinkos |

## VM Configuration

| Setting | Value |
|---------|-------|
| Disk size | 80 GiB |
| RAM | 12G |
| CPUs | 4 |

## Full Layer Stack

1. `aurora-dx-nvidia-open` (external bootc base — full KDE desktop with NVIDIA)
2. `sshd` — SSH server for remote access
3. `ov-full` — ov CLI + virtualization + encrypted storage + console
4. `golang` — Go compiler

## Quick Start

```bash
# Enable in image.yml first (remove enabled: false)
ov image build aurora
ov vm build aurora --type qcow2
ov vm create aurora --ram 12G --cpus 4
ov vm start aurora
ov vm ssh aurora
```

## Key Layers

- `/ov-coder:sshd` — SSH access
- `/ov-foundation:ov` — ov binary in container
- `/ov-coder:golang` — Go compiler

## Related Images

- `/ov-foundation:githubrunner` — another image with ov-full (enabled)

## When to Use This Skill

**MUST be invoked** when the task involves the aurora bootc image or Aurora DX VM deployment.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
