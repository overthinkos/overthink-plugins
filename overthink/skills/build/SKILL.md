---
name: build
description: |
  Building container images with ov build and task build:* commands.
  Use when building images, pushing to registries, merging layers,
  or configuring build caches.
---

# Build - Building Container Images

## Overview

`ov build` generates Containerfiles from `images.yml` and layer definitions, then builds images sequentially in dependency order using the configured build engine (Docker or Podman). Task wrappers in `taskfiles/Build.yml` provide convenient entry points.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build ov CLI | `task build:ov` | Compile Go binary to `bin/ov` |
| Install ov | `task build:install` | Build and install to `~/.local/bin` |
| Build all images | `task build:all` | Build all images in dependency order |
| Build specific image | `task build:local -- <image>` | Build single image for host platform |
| Build and push | `task build:push` | Build all platforms and push to registry |
| Merge layers | `task build:merge -- <image>` | Post-build layer optimization |
| Build ISO | `task build:iso -- <image>` | Build ISO disk image (bootc) |
| Build QCOW2 | `task build:qcow2 -- <image>` | Build QCOW2 VM image (bootc) |
| Build RAW | `task build:raw -- <image>` | Build RAW disk image (bootc) |

## ov build Commands

```bash
ov build [image...]                         # Build for local platform
ov build --push [image...]                  # Build for all platforms and push
ov build --platform linux/amd64 [image...]  # Specific platform
ov build --cache registry|gha [image...]    # Enable build cache
```

## Build Flow

1. Run `ov generate` internally (produces Containerfiles in `.build/`)
2. Resolve runtime config to get build engine (`engine.build`)
3. Get image build order from `ResolveImageOrder()`
4. Filter to requested images (and their base dependencies)
5. For each image: `<engine> build -f .build/<image>/Containerfile -t <tags> .`
6. After all builds: `ov merge --all` (if `merge.auto` enabled, skipped for `--push`)

## Build Cache

| Mode | Backend | Use Case |
|------|---------|----------|
| `registry` | `<registry>/cache:<image>` | Production CI/CD |
| `gha` | GitHub Actions cache | GitHub Actions workflows |

```bash
ov build --cache registry my-image
OV_BUILD_CACHE=gha ov build             # Via environment variable
```

## Push Mode

- **Docker:** `docker buildx build --push` for multi-platform builds
- **Podman:** `podman build --manifest` + `podman manifest push`

## Layer Merging

Post-build optimization that merges consecutive small layers:

```bash
ov merge <image> --dry-run    # Preview what would be merged
ov merge <image>              # Merge small layers
ov merge --all                # Merge all images with merge.auto enabled
ov merge <image> --max-mb 512 # Custom threshold
```

Configure in images.yml:

```yaml
defaults:
  merge:
    auto: true     # Auto-merge after builds
    max_mb: 128    # Max merged layer size (MB)
```

## Engine Configuration

```bash
ov config set engine.build docker   # or podman
ov config set engine.run docker     # or podman
```

## Host Bootstrap (First Time)

Requires: `task`, `go`, `docker` (or `podman`).

```bash
task setup:all       # Build ov CLI + create builder image
task build:all       # Generate + build + merge all images
```

## Common Workflows

### Build a Single Image

```bash
task build:local -- my-app
```

### Rebuild After Layer Changes

```bash
task build:local -- my-app
# Only changed layers are rebuilt (Docker layer cache)
```

### Push to Registry

```bash
# Authenticate first
docker login ghcr.io
# or: podman login ghcr.io

task build:push
```

## Troubleshooting

### "ov not found"

Run `task build:ov` first to compile the CLI.

### Build Fails with Missing Base

Build base images first. `ov build` handles dependency ordering automatically, but if building a single image, its base must already exist.

### Cache Miss

First build on a new machine won't have cache. Use `--cache registry` to pull from registry cache if available.

## Cross-References

- `/overthink:layer` -- Layer definitions that get built
- `/overthink:image` -- Image definitions in images.yml
- `/overthink:deploy` -- Deploying built images
- `/overthink-dev:generate` -- Understanding generated Containerfiles
- Source: `ov/build.go`, `ov/merge.go`, `taskfiles/Build.yml`

## When to Use This Skill

Use when the user asks about:

- Building images (`ov build`, `task build:*`)
- Pushing images to registries
- Build caches (registry, GitHub Actions)
- Layer merging and optimization
- Setting up the build environment
- "How do I build X?"
