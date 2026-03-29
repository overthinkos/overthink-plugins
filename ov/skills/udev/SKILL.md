---
name: udev
description: |
  MUST be invoked before any work involving: GPU device access rules, ov udev commands, udev rule management, or container GPU troubleshooting.
---

# Udev - GPU Device Access Rules

## Overview

Manage udev rules that grant rootless containers access to GPU devices. Without these rules, DRM card nodes and AMD KFD devices may not be accessible to non-root users, blocking GPU features like NVENC encoding and ROCm compute.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Show status | `ov udev status` | GPU devices, groups, rule status, fix suggestions |
| Print rules | `ov udev generate` | Print udev rule content to stdout |
| Install rules | `ov udev install` | Write rules file + reload udev (requires sudo) |
| Remove rules | `ov udev remove` | Delete rules file + reload udev (requires sudo) |

## What the Rules Do

Rules are written to `/etc/udev/rules.d/99-ov-container-access.rules`:

```
# Card nodes: render group access for NVENC hardware encoding
SUBSYSTEM=="drm", KERNEL=="card[0-9]*", GROUP="render", MODE="0660"
# AMD KFD: ROCm compute access
SUBSYSTEM=="kfd", KERNEL=="kfd", GROUP="render", MODE="0660"
```

- **DRM card nodes** (`/dev/dri/card*`) — set group to `render`, mode `0660`. Render nodes (`renderD*`) are already world-accessible; card nodes need explicit rules for NVENC
- **AMD KFD** (`/dev/kfd`) — set group to `render`, mode `0660`. Required for ROCm GPU compute

Rootless Podman's user namespace mapping prevents DRM master abuse, making this safe.

## Status Output

`ov udev status` shows:

```
GPU Devices:
  /dev/dri/card0           nvidia     root:render  0660  OK
  /dev/dri/renderD128      nvidia     root:render  0666  OK

User Groups:
  video:    yes
  render:   yes

Udev Rules:
  /etc/udev/rules.d/99-ov-container-access.rules: installed

Status: OK — GPU device access available for containers
```

If problems are detected, it prints specific fix commands.

## Install Workflow

```bash
ov udev status              # Check current state
ov udev install              # Install rules (prompts for sudo)
```

Install writes the rules file, then runs:
1. `sudo udevadm control --reload-rules`
2. `sudo udevadm trigger --subsystem-match=drm`
3. `sudo udevadm trigger --subsystem-match=kfd`

Idempotent: skips if rules are already up to date.

## Prerequisites

- User must be in the `render` group for GPU access
- AMD GPU users also need the `video` group
- `ov udev status` shows exact `usermod` commands if groups are missing

## Cross-References

- `/ov:doctor` — hardware detection and dependency checks (includes GPU)
- `/ov-layers:nvidia` — NVIDIA GPU runtime layer
- `/ov-layers:rocm` — AMD ROCm GPU compute layer
- `/ov-layers:cuda` — CUDA toolkit layer

## Source

`ov/udev.go`.

## When to Use This Skill

**MUST be invoked** when the task involves GPU device access, udev rules, or troubleshooting container GPU passthrough. Invoke this skill BEFORE reading source code or launching Explore agents.
