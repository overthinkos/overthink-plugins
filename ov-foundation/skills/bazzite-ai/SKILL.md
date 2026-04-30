---
name: bazzite-ai
description: |
  Bazzite NVIDIA bootc image with dev tools, CUDA, Kubernetes, Docker, and desktop apps.
  Currently disabled. Enable in image.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the bazzite-ai image.
---

# bazzite-ai

Bootc VM image based on Universal Blue's Bazzite (gaming-focused) with NVIDIA drivers, comprehensive dev tools, CUDA, Kubernetes, Docker, and desktop applications.

## Image Properties

| Property | Value |
|----------|-------|
| Base | ghcr.io/ublue-os/bazzite-nvidia-open:stable-43.20260120.1 |
| Bootc | true |
| Platforms | linux/amd64 |
| Status | **disabled** (set `enabled: true` in image.yml) |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `bazzite-nvidia-open` (external bootc base — gaming desktop with NVIDIA)
2. Agent forwarding: `agent-forwarding` (gnupg + direnv + ssh-client)
3. Build & dev tools: `build-toolchain`, `language-runtimes`, `dev-tools`
3. Infrastructure: `ujust`, `github-actions`, `virtualization`, `kubernetes`, `docker-ce`
4. Cloud: `google-cloud`, `devops-tools`, `grafana-tools`
5. GPU/ML: `cuda`
6. Desktop: `desktop-apps`, `vscode`, `typst`, `copr-desktop`
7. VR: `vr-streaming`
8. OS config: `os-config`, `os-system-files`

## Key Layers (18 total)

- `/ov-coder:build-toolchain` — gcc, cmake, ninja
- `/ov-foundation:cuda` — NVIDIA CUDA toolkit
- `/ov-coder:kubernetes-layer` — kubectl + Helm
- `/ov-coder:docker-ce` — Docker CE + buildx
- `/ov-foundation:vscode` — Visual Studio Code
- `/ov-foundation:os-config` — Bootc system configuration

## Quick Start

```bash
# Enable in image.yml first (remove enabled: false)
ov image build bazzite-ai
ov vm build bazzite-ai --type qcow2
ov vm create bazzite-ai
ov vm start bazzite-ai
```

## Known Issues

- Base image `terra-mesa.repo` has corrupt zchunk metadata. The generator auto-removes it (`rm -f /etc/yum.repos.d/terra-mesa.repo`).

## Related Images
- `/ov-foundation:aurora` — sibling Universal Blue bootc image with ov toolchain
- `/ov-openclaw:openclaw-browser-bootc` — sibling fedora-bootc image

## Related Commands
- `/ov-advanced:vm` — build and run as a bootc VM
- `/ov-build:build` — produce the bootc image

## When to Use This Skill

**MUST be invoked** when the task involves the bazzite-ai bootc image or the Bazzite-based AI workstation VM.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
