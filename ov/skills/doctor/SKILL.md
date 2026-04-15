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

- **Secret backend availability** — keyring, kdbx, or config. Reports which backend is active and whether it probed healthy.
- **Config file permissions** — warns if `~/.config/ov/config.yml` is not `0600`.
- **Plaintext credential count** — warns if `> 0` plaintext entries are in `config.yml` (suggests `ov settings migrate-secrets`).
- **Secret Service collections** (since 2026-04) — iterates the Secret Service provider's collections and reads the `Label` property on each. A *broken* collection is one whose `org.freedesktop.DBus.Properties.Get` returns `NoSuchObject` or a DBus I/O error — the hallmark of KeePassXC FdoSecrets stubs or a corrupt keyring. Status is `CheckOK` when all collections respond, `CheckWarning` when any are broken (ov iterates past them automatically — see `/ov:enc`). The `Detail` field names the broken path(s) so the user can act on them (KeePassXC → Tools → Settings → Secret Service Integration → Exposed Databases).
- **Keyring index consistency** (since 2026-04) — cross-checks the `keyring_keys` shadow index in `config.yml` against the live Secret Service via `findItemAnyCollection`. For every indexed `service/key` entry, looks it up through the iteration-capable read path. Status is `CheckOK` if `N/N` indexed keys resolve, `CheckWarning` with the stale entries listed otherwise. Remediation hint: `ov secrets set <service> <key>` to re-store, or prune the shadow index.

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

AMD GPU detection also reports the GFX version (e.g., `gfx 11.0.0`) from KFD topology nodes and sets `HSA_OVERRIDE_GFX_VERSION` accordingly.

**DRINODE auto-detection:** `ov` automatically finds the first `/dev/dri/renderD*` device and injects it as `DRINODE` and `DRI_NODE` environment variables into `ov config`, `ov start`, and `ov shell` sessions. This ensures GPU render node selection is consistent across all operations without manual configuration. The detection is centralized in `ov/devices.go` (`DetectedDevices.RenderNode`); the injection is centralized in `appendAutoDetectedEnv()` in the same file.

**Why centralized:** before commit `8f6f322`, DRINODE injection was scattered across 10 separate call sites across `ov`'s source tree — one in `config_image.go`, one in `shell.go`, one in `start.go`, and 7 more in various layer-authored scripts. This led to drift: a fix applied to `ov config` wouldn't reach `ov shell`, and `DRINODE=/dev/dri/renderD129` hardcoded in some selkies scripts masked the auto-detection entirely. The consolidation into `appendAutoDetectedEnv()` means `/ov:config`, `/ov:start`, and `/ov:shell` all produce the identical env set, and `/ov-layers:nvidia` + `/ov-layers:rocm` stopped shipping hardcoded render nodes in their layer.yml.

**Disabling auto-detection:** Pass `--no-autodetect` to `ov config` to skip all of DRINODE, DRI_NODE, and HSA_OVERRIDE_GFX_VERSION injection. Useful when you want to set these values explicitly or test a layer without host device dependence. See `/ov:config` flag table.

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
- `/ov:config` — `engine.build`, `engine.run`, `secret_backend` settings, `--no-autodetect` flag, DRINODE injection via `appendAutoDetectedEnv()`
- `/ov:enc` — credential lookup path behind the Secret Service collection + keyring-index checks; iteration-capable ssClient; broken-collection troubleshooting
- `/ov:secrets` — `ov secrets set/list/prune` commands referenced by the keyring-index remediation hint
- `/ov:settings` — `keyring_collection_label`, `secret_backend`, and other runtime config keys surfaced by the Secret Storage checks
- `/ov:shell` — auto-detected env vars (DRINODE, DRI_NODE, HSA_OVERRIDE_GFX_VERSION) injected via the same `appendAutoDetectedEnv()` path
- `/ov:start` — same auto-injection path at service-start time
- `/ov-layers:nvidia` — NVIDIA GPU runtime support + DRINODE Auto-Injection section
- `/ov-layers:rocm` — AMD ROCm runtime support + DRINODE/HSA_OVERRIDE_GFX_VERSION auto-detect table
- `/ov-layers:selkies` — Primary consumer of DRINODE for VAAPI H.264 encode

## Source

`ov/doctor.go`.

## When to Use This Skill

Use when the user asks about:
- Host dependency checks or setup verification
- GPU hardware detection
- Whether a system is ready for ov operations
- The `ov doctor` command
