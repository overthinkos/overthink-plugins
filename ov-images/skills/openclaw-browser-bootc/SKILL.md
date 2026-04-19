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
| Base | quay.io/fedora/fedora-bootc:43 |
| Bootc | true |
| Layers | agent-forwarding, bootc-base, openclaw, pipewire, wayvnc, chrome-sway |
| Platforms | linux/amd64 |
| Ports | 18789 (gateway), 5900 (VNC), 9222 (CDP) |
| Status | **disabled** (set `enabled: true` in image.yml) |
| Registry | ghcr.io/overthinkos |

## Known latent bug — missing `distro:` declaration

This image's `image.yml` entry does **not** declare `distro:`. Because `base: "quay.io/fedora/fedora-bootc:43"` is an external URL (not the name of another `image.yml` entry), the generator resolves `Distro` to `null`, which short-circuits the install_template's Phase-2 branch — **no layer `rpm:` install RUNs are emitted**. The image will build cleanly but every layer's declarative `rpm:` packages are missing; only `cmd: dnf install …` tasks survive.

This hasn't tripped because the image is `enabled: false`. Before enabling, add:

```yaml
openclaw-browser-bootc:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  distro: ["fedora:43", fedora]   # ← add this
  ...
```

The same latent bug affects `/ov-images:bazzite-ai` and `/ov-images:aurora` (both external ublue bases). See `/ov:image` "External Bases Require Explicit `distro:`" for the full mechanism; `/ov-images:selkies-desktop-bootc` is the canonical working reference.

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
# Enable in image.yml first (remove enabled: false)
ov image build openclaw-browser-bootc
ov vm build openclaw-browser-bootc --type qcow2
ov vm create openclaw-browser-bootc --ram 4G --cpus 2
ov vm start openclaw-browser-bootc
ov vm ssh openclaw-browser-bootc -p 2222
```

## Key Layers

- `/ov-layers:bootc-base` — SSH + guest agent
- `/ov-layers:openclaw` — AI gateway
- `/ov-layers:chrome-sway` — Chrome in Sway

## Related Images

- `/ov-images:openclaw-sway-browser` — container variant (enabled)
- `/ov-images:openclaw-ollama-sway-browser` — with Ollama LLM (enabled)
- `/ov-images:selkies-desktop-bootc` — sibling bootc image with `distro:` correctly declared; follow its pattern when enabling this one
- `/ov-images:bazzite-ai`, `/ov-images:aurora` — share the same latent `distro:` bug

## Related Skills

- `/ov:image` — external-base `distro:` requirement explanation
- `/ov:vm` — VM lifecycle, `/dev:/dev` mount, `vm.ssh_port` plumbing, bootc-VM caveats
- `/ov-layers:bootc-base` — the bootc composition layer pulled in first
- `/ov-layers:bootc-config` — bootc boot wiring (tty1 autologin, graphical target, systemd-user supervisord)

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-browser-bootc VM image or bootc-based OpenClaw deployment.
