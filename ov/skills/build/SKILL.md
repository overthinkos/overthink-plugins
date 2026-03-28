---
name: build
description: |
  MUST be invoked before any work involving: building container images, ov build command, pushing to registries, merging layers, build caches, or Containerfile generation.
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

## Format Config (distro.yml / builder.yml)

Containerfile generation is driven by declarative YAML config files referenced in `images.yml`:

```yaml
defaults:
  format_config:
    distro: distro.yml    # Distro bootstrap + package format definitions
    builder: builder.yml  # Multi-stage builder definitions (pixi, npm, cargo, aur)
```

- **`distro.yml`** — Defines per-distro bootstrap commands (package manager setup, cache mounts, repo management) and package format templates (how `rpm:`, `pac:`, `deb:` sections in layer.yml become `RUN` steps). Each format has `install`, `repos`, `copr`, `modules`, and `options` templates.
- **`builder.yml`** — Defines multi-stage builder patterns (pixi, npm, cargo, aur). Each builder has `build_stage` and `copy_stage` templates that generate the appropriate `FROM builder AS ...` and `COPY --from=...` steps.

Both files use Go `text/template` syntax with access to layer config data. Source: `ov/format_config.go` (loading), `ov/format_template.go` (rendering).

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

### Inline Merge

Images are merged immediately after building, before their children are built. Child images inherit a merged (fewer-layer) base, producing smaller final images. Both local and push builds merge inline. The `mergeAfterBuild()` function handles this -- it checks `merge.auto` on the image config and runs merge if enabled.

For filtered builds (`ov build <image>`), only the built images are merged. For full builds (`ov build`), merge runs after each dependency level completes.

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

### RPM Conflict: ffmpeg-free vs negativo17

If a build fails with `conflicting requests` involving `libavcodec-free` vs `libavcodec` (epoch 1), the layer is trying to install `ffmpeg-free` (Fedora) in an image that has negativo17's `ffmpeg-libs` (via cuda layer). Fix: change `ffmpeg-free` to `ffmpeg` in the layer's `rpm.packages` and add the `fedora-multimedia` repo from negativo17. See the `immich` layer for the correct pattern.

### YAML Unmarshal Error on layer.yml

If you see `cannot unmarshal !!str ... into int` or similar YAML parsing errors on layer fields, the installed `ov` binary is likely stale. Rebuild with `task build:install` or `cp bin/ov ~/.local/bin/ov`. Verify with `ov validate`.

## Cross-References

- `/ov:layer` -- Layer definitions that get built
- `/ov:image` -- Image definitions in images.yml
- `/ov:validate` -- Validating before building
- `/ov:vm` -- Building bootc disk images (`ov vm build`)
- `/ov:config` -- Engine configuration

## When to Use This Skill

**MUST be invoked** when the task involves building images, pushing to registries, build caches, or layer merging. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Typically first in a lifecycle chain.
Next step: `/ov:deploy` (quadlet setup, tunnels) → `/ov:service` (start and manage).
