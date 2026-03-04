---
name: image
description: |
  Image composition: defining and configuring container images in images.yml.
  Use when working with images.yml, image inheritance, defaults, platforms,
  builder configuration, or understanding the image dependency graph.
---

# Image - Image Composition

## Overview

An **image** is a named build target in `images.yml`. Images compose layers into container images with configurable defaults, inheritance chains, platform targets, and builder configurations. The `ov` CLI resolves dependencies, generates Containerfiles, and builds images in the correct order.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `ov list images` | Images from images.yml |
| List build targets | `ov list targets` | Build targets in dependency order (includes auto-intermediates) |
| Inspect image | `ov inspect <image>` | Print resolved config as JSON |
| Inspect field | `ov inspect <image> --format <field>` | Print single field (tag, base, layers, ports, etc.) |
| Validate | `ov validate` | Check images.yml + layers |

## images.yml Structure

```yaml
defaults:
  registry: ghcr.io/overthinkos
  tag: auto                    # CalVer: YYYY.DDD.HHMM
  platforms:
    - linux/amd64
    - linux/arm64
  pkg: rpm
  merge:
    auto: false
    max_mb: 128
  builder: fedora-builder

images:
  fedora:
    base: "quay.io/fedora/fedora:43"
    pkg: rpm

  fedora-builder:
    base: fedora
    layers:
      - pixi
      - nodejs
      - build-toolchain

  my-app:
    base: fedora
    layers:
      - supervisord
      - traefik
      - my-service
    ports:
      - "8080:8080"
```

## Inheritance Chain

Every setting resolves through: **image -> defaults -> hardcoded fallback** (first non-null wins).

| Field | Default | Description |
|-------|---------|-------------|
| `enabled` | `true` | Set `false` to disable (skipped by generate, validate, list) |
| `base` | `quay.io/fedora/fedora:43` | External OCI image or name of another image |
| `bootc` | `false` | Adds `bootc container lint`, enables disk image builds |
| `platforms` | `["linux/amd64", "linux/arm64"]` | Target architectures |
| `tag` | `"auto"` | Image tag. `"auto"` for CalVer |
| `registry` | `""` | Container registry prefix |
| `pkg` | `"rpm"` | Package manager: `"rpm"` or `"deb"` |
| `layers` | (required) | Layer list (image-specific, not inherited) |
| `ports` | `[]` | Runtime port mappings (`"host:container"` or `"port"`) |
| `user` | `"user"` | Username for non-root operations |
| `uid` | `1000` | User ID |
| `gid` | `1000` | Group ID |
| `merge` | `null` | Layer merge settings |
| `aliases` | `[]` | Command aliases |
| `builder` | `""` | Builder image name |
| `bind_mounts` | `[]` | Bind mount declarations (image-level only, not inherited) |

## Builder Image

A dedicated image used as the base for all pixi, npm, and cargo multi-stage build stages.

```yaml
defaults:
  builder: fedora-builder

images:
  fedora-builder:
    base: fedora
    layers:
      - pixi
      - nodejs
      - build-toolchain
```

Resolution: **image.builder -> defaults.builder -> ""**. Builder dependency is conditional -- `ImageNeedsBuilder()` checks whether layers have pixi manifests, `package.json`, or `Cargo.toml`.

## Internal Base Images

When `base` references another image in `images.yml`, the generator resolves it to the full registry/tag and creates a build dependency. The referenced image must be built first.

```yaml
images:
  fedora:
    base: "quay.io/fedora/fedora:43"

  my-app:
    base: fedora        # References fedora image above
    layers: [my-layer]
```

## Intermediate Images

When multiple images share the same base and common layer prefixes, `ov` auto-generates intermediate images at branch points to maximize cache reuse.

```
fedora (external)
  -> fedora-supervisord (auto: pixi + python + supervisord)
     -> fedora-test (adds: traefik, testapi)
     -> openclaw (adds: nodejs, openclaw)
```

Auto-intermediates are marked with `Auto: true` and appear in `ov list targets`.

## Versioning

CalVer: `YYYY.DDD.HHMM` (year, day-of-year, UTC time). Computed once per `ov generate`.

| `tag` value | Generated tag(s) |
|-------------|-----------------|
| `"auto"` | `YYYY.DDD.HHMM` + `latest` |
| `"nightly"` | `nightly` only |
| `"1.2.3"` | `1.2.3` only |

Override: `ov generate --tag <value>`.

## Common Workflows

### Add a New Image

Add an entry to `images.yml` with `base` and `layers`, then build:

```bash
# Edit images.yml
# Then:
task build:local -- my-new-image
```

### Layer Images (inheritance)

Set `base` to another image name:

```yaml
images:
  nvidia:
    base: fedora
    layers: [cuda]
    platforms: [linux/amd64]

  ml-workstation:
    base: nvidia
    layers: [python-ml, jupyter]
```

### Disable an Image

```yaml
images:
  experimental:
    enabled: false
    base: fedora
    layers: [experimental-layer]
```

## Cross-References

- `/overthink:layer` -- Layer definitions that compose into images
- `/overthink:build` -- Building the defined images
- `/overthink:deploy` -- Deploying built images (quadlet, bootc)

## When to Use This Skill

Use when the user asks about:

- Editing or understanding `images.yml`
- Adding new images or image inheritance
- Platform configuration, registry settings
- Builder image setup
- Image defaults and field resolution
- Intermediate images and cache optimization
- "How do I compose images from layers?"
