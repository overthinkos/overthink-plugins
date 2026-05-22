---
name: openclaw-browser-bootc
description: |
  Bootc VM image with OpenClaw gateway, Chrome, VNC, and PipeWire.
  Currently disabled. Enable in image.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the openclaw-browser-bootc image.
---

# openclaw-browser-bootc

Bootable container (bootc) VM image with OpenClaw AI gateway, Chrome browser, VNC access, and PipeWire audio.

## Image Properties

| Property | Value |
|----------|-------|
| Location | `overthinkos/bootc` submodule (`image/bootc`) — composed by `@github` ref |
| Base | quay.io/fedora/fedora-bootc:43 |
| Bootc | true |
| Distro | `fedora:43`, `fedora` (declared — external base inherits no distro tags) |
| Layers | agent-forwarding, bootc-base, openclaw, pipewire, wayvnc, chrome-sway |
| Platforms | linux/amd64 |
| Ports | 18789 (gateway), 5900 (VNC), 9222 (CDP) |
| Status | **disabled** (build with `--include-disabled`) |
| Registry | ghcr.io/overthinkos |

## External base requires explicit `distro:` (declared)

`base: "quay.io/fedora/fedora-bootc:43"` is an external URL, so it inherits no distro tags — without `distro:` the generator short-circuits the install_template's Phase-2 branch and emits **zero** layer `rpm:` installs. This image now declares `distro: ["fedora:43", fedora]` (added during the 2026-05 bootc-submodule extraction), matching the `/ov-selkies:selkies-desktop-bootc` working pattern. The sibling ublue images `/ov-distros:bazzite` and `/ov-distros:aurora` declare it too. See `/ov-image:image` "External Bases Require Explicit `distro:`" for the mechanism.

## VM Configuration

| Setting | Value |
|---------|-------|
| SSH port | 2222 |
| Disk size | 20 GiB |
| RAM | 4G |
| CPUs | 2 |

## Full Layer Stack

1. `fedora-bootc:43` (external bootc base)
2. `bootc-base` — sshd + guest agent + bootc config
3. `openclaw` — AI gateway
4. `pipewire` — audio server
5. `wayvnc` — VNC server
6. `chrome-sway` — Chrome in Sway compositor

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 18789 | OpenClaw gateway | HTTP |
| 5900 | VNC | TCP |
| 9222 | Chrome DevTools | HTTP |

## Quick Start

```bash
# Built from the bootc submodule; all images ship enabled:false.
ov -C image/bootc image build openclaw-browser-bootc --include-disabled
ov -C image/bootc vm build openclaw-browser-bootc-bootc --transport containers-storage
ov -C image/bootc vm create openclaw-browser-bootc-bootc --ram 4G --cpus 2
ov -C image/bootc vm start openclaw-browser-bootc-bootc
ov -C image/bootc vm ssh openclaw-browser-bootc-bootc
```

## Key Layers

- `/ov-distros:bootc-base` — SSH + guest agent
- `/ov-openclaw:openclaw` — AI gateway
- `/ov-selkies:chrome-sway` — Chrome in Sway

## Related Images

- `/ov-openclaw:openclaw-sway-browser` — container variant (enabled)
- `/ov-openclaw:openclaw-ollama-sway-browser` — with Ollama LLM (enabled)
- `/ov-selkies:selkies-desktop-bootc` — sibling bootc image (canonical worked example); same `image/bootc` submodule
- `/ov-distros:bazzite`, `/ov-distros:aurora` — sibling ublue bootc images; same `image/bootc` submodule

## Related Skills

- `/ov-image:image` — external-base `distro:` requirement explanation
- `/ov-vm:vm` — VM lifecycle, `/dev:/dev` mount, `vm.ssh_port` plumbing, bootc-VM caveats
- `/ov-distros:bootc-base` — the bootc composition layer pulled in first
- `/ov-distros:bootc-config` — bootc boot wiring (tty1 autologin, graphical target, systemd-user supervisord)

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-browser-bootc VM image or bootc-based OpenClaw deployment.

## Related

- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
