---
name: fedora-nonfree
description: |
  Fedora with RPM Fusion non-free repositories enabled. Base for images
  needing codec or proprietary package support like immich.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora-nonfree image.
---

# fedora-nonfree

Fedora base with RPM Fusion free and non-free repositories enabled.

**Defined in `base.yml`.** Lives in the main repo's combined `base.yml`
(single source of truth), alongside `/charly-distros:fedora` +
`/charly-distros:fedora-builder`, and is imported under the `charly` namespace by the
`overthinkos/fedora` submodule (referenced as `charly.fedora-nonfree`). It lives in
main because in-main consumers (`/charly-immich:immich`,
`/charly-distros:nvidia`) base on it. Its `rpmfusion`
layer is pinned as a github ref so the same definition resolves in both main and
the submodule.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | rpmfusion |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (quay.io/fedora/fedora:43)
2. `rpmfusion` — RPM Fusion free + non-free repos

## Quick Start

```bash
charly box build fedora-nonfree
charly shell fedora-nonfree
```

## Key Layers

- `/charly-distros:rpmfusion` — RPM Fusion repository configuration

## Derived Images

- `/charly-immich:immich` — photo management with codec support

## Related Images

- `/charly-distros:fedora` — parent base (without non-free repos)

## Verification

After `charly box build`:
- `charly box list` — image appears in list
- `charly shell fedora-nonfree` — interactive shell works

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-nonfree image, RPM Fusion support, or non-free codec packages. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/charly-image:image` — image family umbrella (`image:` entries in `charly.yml`, build/validate/inspect/list)
- `/charly-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
