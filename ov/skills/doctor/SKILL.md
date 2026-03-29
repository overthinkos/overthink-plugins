---
name: doctor
description: |
  Host dependency checker and hardware detector.
  Use when diagnosing host setup, checking dependencies, or verifying GPU detection.
---

# Doctor - Host Dependency Check

## Overview

`ov doctor` checks all host dependencies grouped by feature area, probes for GPU and device hardware, and reports a summary. Use it to diagnose missing tools, verify GPU setup, or check if a host is ready for ov operations.

## Usage

```bash
ov doctor              # Human-readable output
ov doctor --json       # Machine-readable JSON (DoctorOutput struct)
```

## Check Groups

Dependencies are organized into groups. Required groups cause a non-zero exit if all checks fail.

### Container Engine (required, OR-logic)

At least one must be installed:
- `docker`
- `podman`

### Build Infrastructure

- `go` — required to build ov from source
- `git`
- `docker buildx` — only checked if docker is available

### Service Management (quadlet mode)

- `systemctl`
- `podman` (for quadlet)

### Virtual Machines

- `qemu-system-x86_64` (or arch-specific variant)
- `qemu-img`
- `virtiofsd` — checks PATH + `/usr/lib/virtiofsd` + `/usr/libexec/virtiofsd`
- `virsh`
- `ssh`
- libvirt session socket

### Encrypted Storage

- `gocryptfs`
- `fusermount3`
- `systemd-ask-password`

### Secret Storage

- Secret backend availability (keyring, kdbx, or config)
- Config file permissions (warns if not `0600`)
- Plaintext credential count (warns if > 0)

### Tunnels

- `tailscale`
- `cloudflared`

### Merge & Registry

- `skopeo`

### Shell & TTY

- `script`

### Podman Machine (conditional)

Only shown if podman is installed:
- `gvproxy` — checks PATH + `/usr/libexec/podman/gvproxy` + `/usr/lib/podman/gvproxy`

## Hardware Detection

Probes GPU and device hardware, reports what flags containers will receive:

| Device | Description | Container flag |
|--------|-------------|---------------|
| NVIDIA GPU | CUDA-capable GPU | `--gpus all` or CDI device |
| AMD GPU | ROCm compute | `--group-add keep-groups` |
| `/dev/dri/renderD*` | GPU render node | `--device /dev/dri/renderD128` |
| `/dev/kfd` | AMD Kernel Fusion Driver | `--device /dev/kfd` |
| `/dev/kvm` | KVM virtualization | `--device /dev/kvm` |
| `/dev/vhost-net` | vhost network acceleration | `--device /dev/vhost-net` |
| `/dev/vhost-vsock` | VM socket communication | `--device /dev/vhost-vsock` |
| `/dev/fuse` | FUSE filesystem | `--device /dev/fuse` |
| `/dev/net/tun` | TUN/TAP network device | `--device /dev/net/tun` |
| `/dev/hwrng` | Hardware RNG | `--device /dev/hwrng` |

AMD GPU detection also reports the GFX version (e.g., `gfx1030`).

## Output Format

Human-readable output uses symbols:
- `[+]` — installed / detected
- `[-]` — missing
- `[!]` — warning (installed but with caveats)
- `[ ]` — not present (hardware, neutral)

Each check shows the binary path and version when available, or an install hint when missing. Install hints are distro-aware (suggests `pacman`, `dnf`, `apt` as appropriate).

## JSON Output

`ov doctor --json` emits a `DoctorOutput` struct with:
- `system` — detected distro info
- `groups` — all check groups with individual results
- `hardware` — GPU flags, device list, container flags
- `summary` — counts of installed, missing, warnings, devices

## Cross-References

- `/ov:udev` — install udev rules for GPU device access
- `/ov:config` — `engine.build`, `engine.run`, `secret_backend` settings

## Source

`ov/doctor.go`.

## When to Use This Skill

Use when the user asks about:
- Host dependency checks or setup verification
- GPU hardware detection
- Whether a system is ready for ov operations
- The `ov doctor` command
