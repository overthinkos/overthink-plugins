---
name: openclaw-browser-bootc
description: |
  Bootc VM image with OpenClaw gateway, Chrome, VNC, and PipeWire.
  Currently disabled. Enable in images.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the openclaw-browser-bootc image.
---

# openclaw-browser-bootc

Bootable container (bootc) VM image with OpenClaw AI gateway, Chrome browser, VNC access, and PipeWire audio.

## Image Properties

| Property | Value |
|----------|-------|
| Base | quay.io/fedora/fedora-bootc:43 |
| Bootc | true |
| Layers | bootc-base, openclaw, pipewire, wayvnc, chrome-sway |
| Platforms | linux/amd64 |
| Ports | 18789 (gateway), 5900 (VNC), 9222 (CDP) |
| Status | **disabled** (set `enabled: true` in images.yml) |
| Registry | ghcr.io/overthinkos |

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
# Enable in images.yml first (remove enabled: false)
ov build openclaw-browser-bootc
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

## When to Use This Skill

**MUST be invoked** when the task involves the openclaw-browser-bootc VM image or bootc-based OpenClaw deployment.
