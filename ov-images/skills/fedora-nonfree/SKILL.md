---
name: fedora-nonfree
description: |
  Fedora with RPM Fusion non-free repositories enabled. Base for images
  needing codec or proprietary package support like immich.
  Use when working with non-free packages or the immich image chain.
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
ov build fedora-nonfree
ov shell fedora-nonfree
```

## Key Layers

- `/ov-layers:rpmfusion` — RPM Fusion repository configuration

## Derived Images

- `/ov-images:immich` — photo management with codec support

## Related Images

- `/ov-images:fedora` — parent base (without non-free repos)

## Verification

After `ov build`:
- `ov list` — image appears in list
- `ov shell fedora-nonfree` — interactive shell works

## When to Use This Skill

Use when the user asks about the fedora-nonfree image, RPM Fusion support, or non-free codec packages in containers.
