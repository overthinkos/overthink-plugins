---
name: fedora
description: |
  Base Fedora 43 image. Root of the image hierarchy for all RPM-based images.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora image.
---

# fedora

Root base image built from `quay.io/fedora/fedora:43`. Foundation for all RPM-based Overthink images.

**Defined in the combined `base.yml`.** The Fedora base stack — `fedora`,
`/ov-distros:fedora-builder`, `/ov-distros:fedora-nonfree` — lives in the main
repo's `base.yml` (single source of truth, shared with the arch stack),
flat-imported locally by main AND imported under the `ov` namespace by the
**`overthinkos/fedora`** submodule (mounted at `image/fedora`), which references
them as `ov.fedora` / `ov.fedora-builder`. The base stack lives in main because
fedora is the ecosystem default base (~40 main images root on it,
`fedora-builder` is `defaults.builder`); the Fedora consumer showcase images
(`/ov-coder:fedora-coder`, `/ov-distros:fedora-ov`, `/ov-distros:fedora-test`)
live in the submodule (a single `overthink.yml`, all per-kind entries inlined).
Build with `ov image build fedora` from the main repo.

## Image Properties

| Property | Value |
|----------|-------|
| Base | quay.io/fedora/fedora:43 |
| Layers | (none) |
| Platforms | linux/amd64 |
| Pkg | rpm |
| Registry | ghcr.io/overthinkos |

## Quick Start

```bash
ov image build fedora
ov shell fedora
```

## Derived Images

- `/ov-distros:fedora-nonfree` — adds RPM Fusion repos
- `/ov-distros:fedora-builder` — adds pixi, nodejs, build-toolchain
- `/ov-distros:nvidia` — adds CUDA toolkit
- `/ov-openclaw:openclaw` — adds OpenClaw gateway
- `/ov-distros:githubrunner` — adds GitHub Actions runner

## Verification

After `ov image build`:
- `ov image list` — image appears in list
- `ov shell fedora` — interactive shell works

## Related Images
- `/ov-distros:fedora-nonfree` — adds RPM Fusion repos
- `/ov-distros:fedora-builder` — adds pixi, nodejs, build-toolchain
- `/ov-distros:fedora-ov` — full ov toolchain on Fedora
- `/ov-distros:arch` — pacman-based counterpart base

## Related Commands
- `/ov-build:build` — build the fedora base image
- `/ov-core:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the fedora base image or understanding the image chain. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-image:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
