---
name: fedora
description: |
  Base Fedora 43 image. Root of the image hierarchy for all RPM-based images.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora image.
---

# fedora

Root base image built from `quay.io/fedora/fedora:43`. Foundation for all RPM-based OpenCharly images.

**Defined in the combined `base.yml`.** The Fedora base stack — `fedora`,
`/charly-distros:fedora-builder`, `/charly-distros:fedora-nonfree` — lives in the main
repo's `base.yml` (single source of truth, shared with the arch stack),
flat-imported locally by main AND imported under the `ov` namespace by the
**`overthinkos/fedora`** submodule (mounted at `image/fedora`), which references
them as `ov.fedora` / `ov.fedora-builder`. The base stack lives in main because
fedora is the ecosystem default base (~40 main images root on it,
`fedora-builder` is `defaults.builder`); the Fedora consumer showcase images
(`/charly-coder:fedora-coder`, `/charly-distros:fedora-ov`, `/charly-distros:fedora-test`)
live in the submodule (its `charly.yml` plus per-kind sibling files
(`box.yml`/`pod.yml`/`k8s.yml`), flat-imported via `import:`).
Build with `charly box build fedora` from the main repo.

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
charly box build fedora
charly shell fedora
```

## dnf download tuning (`distro.fedora.dnf`)

`build.yml` `distro.fedora.dnf` writes dnf download-speed knobs to
`/etc/dnf/dnf.conf` during the bootstrap, so they apply to the bootstrap install
AND every per-layer dnf install in this image and its descendants:

```yaml
# build.yml
distro:
  fedora:
    dnf:
      max_parallel_downloads: 10   # concurrent package downloads
      fastestmirror: true          # sort mirrors by measured speed
```

These are **speed-only** — they never change which packages are selected
(`install_weak_deps` stays on the bootstrap `install_cmd`'s
`--setopt=install_weak_deps=False`). The block is a `DnfConfig` on `DistroDef`
and inherits across distro inheritance like the other sub-blocks. Source:
`ov/generate.go:renderDnfConfWrite`.

## Derived Images

- `/charly-distros:fedora-nonfree` — adds RPM Fusion repos
- `/charly-distros:fedora-builder` — adds pixi, nodejs, build-toolchain
- `/charly-distros:nvidia` — adds CUDA toolkit
- `/charly-openclaw:openclaw` — adds OpenClaw gateway
- `/charly-distros:githubrunner` — adds GitHub Actions runner

## Verification

After `charly box build`:
- `charly box list` — image appears in list
- `charly shell fedora` — interactive shell works

## Related Images
- `/charly-distros:fedora-nonfree` — adds RPM Fusion repos
- `/charly-distros:fedora-builder` — adds pixi, nodejs, build-toolchain
- `/charly-distros:fedora-ov` — full charly toolchain on Fedora
- `/charly-distros:arch` — pacman-based counterpart base

## Related Commands
- `/charly-build:build` — build the fedora base image
- `/charly-core:shell` — interactive shell in the base image

## When to Use This Skill

**MUST be invoked** when the task involves the fedora base image or understanding the image chain. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
