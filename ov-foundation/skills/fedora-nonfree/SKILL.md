---
name: fedora-nonfree
description: |
  Fedora with RPM Fusion non-free repositories enabled. Base for images
  needing codec or proprietary package support like immich.
  MUST be invoked before building, deploying, configuring, or troubleshooting the fedora-nonfree image.
---

# fedora-nonfree

Fedora base with RPM Fusion free and non-free repositories enabled.

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
ov image build fedora-nonfree
ov shell fedora-nonfree
```

## Key Layers

- `/ov-foundation:rpmfusion` — RPM Fusion repository configuration

## Derived Images

- `/ov-immich:immich` — photo management with codec support

## Related Images

- `/ov-foundation:fedora` — parent base (without non-free repos)

## Verification

After `ov image build`:
- `ov image list` — image appears in list
- `ov shell fedora-nonfree` — interactive shell works

## When to Use This Skill

**MUST be invoked** when the task involves the fedora-nonfree image, RPM Fusion support, or non-free codec packages. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
