---
name: bazzite
description: |
  Bazzite NVIDIA bootc image with dev tools, CUDA, Kubernetes, Docker, and desktop apps.
  Currently disabled. Enable in box.yml to build.
  MUST be invoked before building, deploying, or troubleshooting the bazzite image.
---

# bazzite

Bootc VM image based on Universal Blue's Bazzite (gaming-focused) with NVIDIA drivers, comprehensive dev tools, CUDA, Kubernetes, Docker, and desktop applications. Lives in the `overthinkos/bootc` submodule (`image/bootc`).

## Image Properties

| Property | Value |
|----------|-------|
| Location | `overthinkos/bootc` submodule (`image/bootc`) — composed by `@github` ref |
| Base | ghcr.io/ublue-os/bazzite-nvidia-open:stable-43.20260120.1 |
| Bootc | true |
| Distro | `fedora:43`, `fedora` (declared — external base inherits no distro tags) |
| Platforms | linux/amd64 |
| Status | enabled |
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

- `/charly-coder:build-toolchain` — gcc, cmake, ninja
- `/charly-distros:cuda` — NVIDIA CUDA toolkit
- `/charly-coder:kubernetes-layer` — kubectl + Helm
- `/charly-coder:docker-ce` — Docker CE + buildx
- `/charly-tools:vscode` — Visual Studio Code
- `/charly-distros:os-config` — Bootc system configuration

## Quick Start

```bash
# Built from the bootc submodule.
charly -C image/bootc image build bazzite
charly -C image/bootc vm build bazzite-bootc --transport containers-storage
charly -C image/bootc vm create bazzite-bootc
charly -C image/bootc vm start bazzite-bootc
```

## Known Issues

- Base image `terra-mesa.repo` has corrupt zchunk metadata. The generator auto-removes it (`rm -f /etc/yum.repos.d/terra-mesa.repo`).

## Related Images
- `/charly-distros:aurora` — sibling Universal Blue bootc image with charly toolchain

## Related Commands
- `/charly-vm:vm` — build and run as a bootc VM
- `/charly-build:build` — produce the bootc image

## When to Use This Skill

**MUST be invoked** when the task involves the bazzite bootc image or the Bazzite-based AI workstation VM.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
