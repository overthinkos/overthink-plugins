---
name: build
description: |
  Building container images with ov build.
  Use when building images, pushing to registries, merging layers,
  or configuring build caches.
---

# Build - Building Container Images

## Overview

`ov build` generates Containerfiles from `images.yml` and layer definitions, then builds images in dependency order using the configured build engine (Docker or Podman). Images at the same dependency level are built in parallel (up to `--jobs` concurrent builds).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build all images | `ov build` | Build all images in dependency order |
| Build specific image | `ov build <image>` | Build single image for host platform |
| Build and push | `ov build --push` | Build all platforms and push to registry |
| Build without cache | `ov build --no-cache` | Disable build cache entirely |
| Merge layers | `ov merge <image>` | Post-build layer optimization |
| Build QCOW2 | `ov vm build <image>` | Build QCOW2 disk image (bootc) |
| Build RAW | `ov vm build <image> --type raw` | Build RAW disk image (bootc) |

## ov build Commands

```bash
ov build [image...]                         # Build for local platform
ov build --push [image...]                  # Build for all platforms and push
ov build --platform linux/amd64 [image...]  # Specific platform
ov build --cache registry [image...]         # Registry cache (read+write)
ov build --cache image [image...]           # Image cache (read-only, default)
ov build --cache gha [image...]             # GitHub Actions cache
ov build --no-cache [image...]              # Disable cache entirely
ov build --jobs N [image...]                # Max concurrent builds per level (default: 4)
```

## Containerfile Generation

`ov build` runs `ov generate` internally. You can also run it standalone to inspect generated Containerfiles:

```bash
ov generate                          # Write .build/ (Containerfiles)
ov generate --tag v1.0.0             # Override CalVer tag
cat .build/my-image/Containerfile    # Inspect generated output
```

## Build Flow

1. Run `ov generate` internally (produces Containerfiles in `.build/`)
2. Resolve runtime config to get build engine (`engine.build`)
3. Resolve image build order (dependency ordering, grouped by level)
4. Filter to requested images (and their base dependencies)
5. For each level: build images in parallel (up to `--jobs` concurrent, default 4)
6. After all builds: `ov merge --all` (if `merge.auto` enabled, skipped for `--push`)

## Build Cache

| Mode | Backend | Use Case |
|------|---------|----------|
| `image` | `<registry>/<image>` | Read-only cache from registry image (default) |
| `registry` | `<registry>/cache:<image>` | Production CI/CD (read + write) |
| `gha` | GitHub Actions cache | CI builds on GitHub Actions |
| `none` | No cache | Same as `--no-cache` |

```bash
ov build --cache registry my-image        # Read+write registry cache
ov build --cache image my-image          # Read-only from registry image
OV_BUILD_CACHE=registry ov build         # Via environment variable
```

## Build Flow Details

**Internal base images** use exact CalVer tags in Containerfiles (`FROM ghcr.io/overthinkos/fedora:2026.46.1415`). This ensures each image references the precise version of its parent. Both Docker and Podman resolve local images before pulling from registry.

## Push Mode

- **Docker:** `docker buildx build --push` for multi-platform builds
- **Podman:** `podman build --manifest` + `podman manifest push`

Podman manifest push uses retry with exponential backoff (3 attempts, 5s/10s/20s delays) to handle transient registry errors (e.g., GHCR 500 errors after long builds).

Source: `ov/build.go` (`retryCmd`).

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

CLI flag `--max-mb` overrides `images.yml`. `auto` is only used by `ov merge --all` to select which images to merge; `ov merge <image>` always merges regardless.

### Algorithm

1. Load image from engine via `<engine> save` -> `tarball.ImageFromPath()`
2. Get compressed sizes via `layer.Size()`
3. Group consecutive layers into groups totaling <= `max_mb`
4. Single-layer "groups" are kept as-is (need 2+ layers to merge)
5. For each merge group: read uncompressed tarballs, deduplicate entries by path (last writer wins), write combined tar into a single new layer
6. Reconstruct image with `mutate.Append()`, preserving OCI history alignment (empty-layer entries for ENV/USER/EXPOSE kept in correct positions)
7. Save via `tarball.WriteToFile()` -> `<engine> load`

Merge is idempotent -- running again after merging shows all layers as `[keep]`. Source: `ov/merge.go`.

## Engine Configuration

```bash
ov config set engine.build docker   # or podman
ov config set engine.run docker     # or podman
```

## Host Bootstrap (First Time)

Requires: `go`, `docker` (or `podman`). Run `bash setup.sh` to download `task`, build the `ov` CLI, then use `ov` directly.

```bash
bash setup.sh        # Download task, build ov CLI
task setup:builder   # Create multi-platform buildx builder
ov build             # Generate + build + merge all images
```

## Common Workflows

### Build a Single Image

```bash
ov build my-app
```

### Rebuild After Layer Changes

```bash
ov build my-app
# Only changed layers are rebuilt (Docker layer cache)
```

### Push to Registry

```bash
# Authenticate first
docker login ghcr.io
# or: podman login ghcr.io

ov build --push
```

## Troubleshooting

### "ov not found"

Run `bash setup.sh` to download task and compile the CLI, or `task build:ov` if task is already installed.

### Build Fails with Missing Base

Build base images first. `ov build` handles dependency ordering automatically, but if building a single image, its base must already exist.

### Cache Miss

First build on a new machine won't have cache. Use `--cache registry` to pull from registry cache if available.

## Cross-References

- `/overthink:layer` -- Layer definitions that get built
- `/overthink:image` -- Image definitions in images.yml
- `/overthink:validate` -- Validating before building
- `/overthink:deploy` -- Deploying built images

## When to Use This Skill

Use when the user asks about:

- Building images (`ov build`)
- Pushing images to registries
- Build caches (registry, GitHub Actions)
- Layer merging and optimization
- Setting up the build environment
- "How do I build X?"
