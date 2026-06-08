---
name: aurora
description: |
  Aurora DX bootc image with NVIDIA, SSH, charly toolchain, and Go.
  Currently disabled. Enable in box.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the aurora image.
---

# aurora

Bootc VM image based on Universal Blue's Aurora DX with NVIDIA drivers, SSH access, full charly toolchain, and Go compiler. Lives in the `overthinkos/bootc` submodule (`image/bootc`).

## Image Properties

| Property | Value |
|----------|-------|
| Location | `overthinkos/bootc` submodule (`image/bootc`) — composed by `@github` ref |
| Base | ghcr.io/ublue-os/aurora-dx-nvidia-open:stable-daily-43.20260305 |
| Bootc | true |
| Distro | `fedora:43`, `fedora` (declared — external base inherits no distro tags) |
| Layers | agent-forwarding, bootc-base, sshd, ov, golang |
| Platforms | linux/amd64 |
| Status | enabled |
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
3. `charly` — the full toolchain: charly CLI + virtualization + encrypted storage + console
4. `golang` — Go compiler

## Quick Start

```bash
# Built from the bootc submodule.
charly -C image/bootc image build aurora
charly -C image/bootc vm build aurora-bootc --transport containers-storage
charly -C image/bootc vm create aurora-bootc --ram 12G --cpus 4
charly -C image/bootc vm start aurora-bootc
charly -C image/bootc vm ssh aurora-bootc
```

## Key Layers

- `/charly-coder:sshd` — SSH access
- `/charly-tools:charly` — charly binary in container
- `/charly-coder:golang` — Go compiler

## Related Images

- `/charly-distros:githubrunner` — another image with the full `charly` toolchain (enabled)

## When to Use This Skill

**MUST be invoked** when the task involves the aurora bootc image or Aurora DX VM deployment.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
