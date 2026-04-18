---
name: build
description: |
  MUST be invoked before any work involving: building container images, ov image build command, pushing to registries, merging layers, build caches, or Containerfile generation.
---

# ov image build -- Building Container Images

Invoked as `ov image build`. See `/ov:image` for the family overview.

## Overview

`ov image build` generates Containerfiles from `images.yml` and layer definitions, then builds images in dependency order using the configured build engine (Docker or Podman). Images at the same dependency level are built in parallel (up to `--jobs` concurrent builds).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build all images | `ov image build` | Build all images in dependency order |
| Build specific image | `ov image build <image>` | Build single image for host platform |
| Build and push | `ov image build --push` | Build all platforms and push to registry |
| Build without cache | `ov image build --no-cache` | Disable build cache entirely |
| Merge layers | `ov image merge <image>` | Post-build layer optimization |

## ov image build Commands

```bash
ov image build [image...]                         # Build for local platform
ov image build --push [image...]                  # Build for all platforms and push
ov image build --platform linux/amd64 [image...]  # Specific platform
ov image build --cache registry [image...]         # Registry cache (read+write)
ov image build --cache image [image...]           # Image cache (read-only, default)
ov image build --cache gha [image...]             # GitHub Actions cache
ov image build --no-cache [image...]              # Disable cache entirely
ov image build --jobs N [image...]                # Max concurrent images per DAG level (default: 4)
ov image build --podman-jobs N [image...]         # Max concurrent stages within a single podman build (default: min(NCPU, 4))
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

`ov image build` runs `ov image generate` internally. You can also run it standalone to inspect generated Containerfiles:

```bash
ov image generate                          # Write .build/ (Containerfiles)
ov image generate --tag v1.0.0             # Override CalVer tag
cat .build/my-image/Containerfile    # Inspect generated output
```

## Build Flow

1. Run `ov image generate` internally (produces Containerfiles in `.build/`)
2. Resolve runtime config to get build engine (`engine.build`)
3. Resolve image build order (dependency ordering, grouped by level)
4. Filter to requested images (and their base dependencies)
5. For each level: build images in parallel (up to `--jobs` concurrent, default 4)
6. After all builds: `ov image merge --all` (if `merge.auto` enabled, skipped for `--push`)

## Parallelism: `--jobs` vs `--podman-jobs`

ov exposes **two** parallelism knobs with distinct meanings:

| Flag | Env var | Default | What it controls |
|---|---|---|---|
| `--jobs N` | `OV_BUILD_JOBS` | `4` | **Outer** concurrency: how many ov-level images to build in parallel within a DAG level (e.g., when `ov image build` rebuilds the whole graph). |
| `--podman-jobs N` | `OV_PODMAN_JOBS` | auto = `min(NCPU, 4)` | **Inner** concurrency: passed to `podman build --jobs N`, controls how many stages of a *single* multi-stage build run concurrently. |

The inner default is **capped at 4** because podman-5.7.x races under high
concurrency in its blob-reuse code path (`storage_dest.go:TryReusingBlobWithOptions`
and `queueOrCommit`). Observed reproducibly on `selkies-desktop` (29-stage
DAG) with `--jobs runtime.NumCPU()` (16 on a 16-core host) and `--cache-from`:
podman SIGABRTs with a core dump in the middle of the blob-reuse path. The cap
narrows the race window enough that the bug has not been observed to fire in
practice. Core dumps are captured by `systemd-coredump` at
`/var/lib/systemd/coredump/core.podman.*.zst` — inspect with `coredumpctl info`
to confirm which build was faulting.

Override the cap if your podman version is known-good (upstream upgrades
may eventually fix it):

```bash
ov image build <image> --podman-jobs 16             # fully parallel stages
OV_PODMAN_JOBS=8 ov update <image> --build    # via env
ov image build <image> --podman-jobs 1              # fully serialised, worst-case debugging
```

Source: `ov/build.go:resolvePodmanJobs` + `podmanJobsDefault`. Covered by
`ov/build_jobs_test.go` (12 unit + integration cases). The outer `--jobs` knob
lives on the `BuildCmd` struct; the inner `--podman-jobs` was added as a
separate field so the two semantics don't get conflated.

## Build Cache

| Mode | Backend | Use Case |
|------|---------|----------|
| `image` | `<registry>/<image>` | Read-only cache from registry image (default) |
| `registry` | `<registry>/cache:<image>` | Production CI/CD (read + write) |
| `gha` | GitHub Actions cache | CI builds on GitHub Actions |
| `none` | No cache | Same as `--no-cache` |

```bash
ov image build --cache registry my-image        # Read+write registry cache
ov image build --cache image my-image          # Read-only from registry image
OV_BUILD_CACHE=registry ov image build         # Via environment variable
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
ov image merge <image> --dry-run         # Preview what would be merged
ov image merge <image>                   # Merge small layers
ov image merge --all                     # Merge all images with merge.auto enabled
ov image merge <image> --max-mb 512      # Custom per-layer threshold
ov image merge <image> --max-total-mb 4096  # Custom total image size limit
```

Configure in images.yml:

```yaml
defaults:
  merge:
    auto: true        # Auto-merge after builds
    max_mb: 128       # Max merged layer size (MB)
    max_total_mb: 0   # Max total image size for merge (0 = no limit)
```

CLI flags `--max-mb` and `--max-total-mb` override `images.yml`. `auto` is only used by `ov image merge --all` to select which images to merge; `ov image merge <image>` always merges regardless. `max_total_mb` controls whether large images skip merging entirely (the merge process decompresses layers in memory). Set to `0` to disable (default), or a positive value like `2048` to cap on low-memory CI runners.

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

For filtered builds (`ov image build <image>`), only the built images are merged. For full builds (`ov image build`), merge runs after each dependency level completes.

## Engine Configuration

```bash
ov settings set engine.build docker   # or podman
ov settings set engine.run docker     # or podman
```

## Host Bootstrap (First Time)

Requires: `go`, `docker` (or `podman`). Run `bash setup.sh` to download `task`, build the `ov` CLI, then use `ov` directly.

```bash
bash setup.sh        # Download task, build ov CLI
task setup:builder   # Create multi-platform buildx builder
ov image build             # Generate + build + merge all images
```

## Common Workflows

### Build a Single Image

```bash
ov image build my-app
```

### Rebuild After Layer Changes

```bash
ov image build my-app
# Only changed layers are rebuilt (Docker layer cache)
```

### Push to Registry

```bash
# Authenticate first
docker login ghcr.io
# or: podman login ghcr.io

ov image build --push
```

## Troubleshooting

### "ov not found"

Run `bash setup.sh` to download task and compile the CLI, or `task build:ov` if task is already installed.

### Build Fails with Missing Base

Build base images first. `ov image build` handles dependency ordering automatically, but if building a single image, its base must already exist.

### Cache Miss

First build on a new machine won't have cache. Use `--cache registry` to pull from registry cache if available.

### RPM Conflict: ffmpeg-free vs negativo17

If a build fails with `conflicting requests` involving `libavcodec-free` vs `libavcodec` (epoch 1), the layer is trying to install `ffmpeg-free` (Fedora) in an image that has negativo17's `ffmpeg-libs` (via cuda layer). Fix: change `ffmpeg-free` to `ffmpeg` in the layer's `rpm.packages` and add the `fedora-multimedia` repo from negativo17. See the `immich` layer for the correct pattern.

### YAML Unmarshal Error on layer.yml

If you see `cannot unmarshal !!str ... into int` or similar YAML parsing errors on layer fields, the installed `ov` binary is likely stale. Rebuild with `task build:install` or `cp bin/ov ~/.local/bin/ov`. Verify with `ov image validate`.

### `--no-cache` does not invalidate intermediate scratch-stage caches

`ov image build --no-cache <image>` and `ov image build --cache none <image>` reliably disable the
cache for the **final image stage**, but in observed behavior they do **not** propagate
to intermediate scratch stages produced by `COPY layers/<x>/ /` instructions
(`[15/25] STEP 2/2: COPY layers/labwc/ /` style). Editing a single file inside a layer
directory and rebuilding with `--no-cache` may still pull the labwc scratch stage from
cache, leaving the new file content out of the rebuilt image.

**Workaround:** force a content-hash bump on the layer's `layer.yml`. The simplest is
adding (or removing) a trailing comment line:

```bash
echo "" >> layers/labwc/layer.yml          # bump content hash
ov image build selkies-desktop                   # now invalidates the labwc scratch stage
git checkout -- layers/labwc/layer.yml     # revert the cosmetic change
```

This was discovered while shipping commit `febb9bd` (labwc autostart race fix): the
edit to `layers/labwc/autostart` did not propagate to the rebuilt image until
`layers/labwc/layer.yml` itself was touched. Two consecutive `--no-cache` rebuilds
produced the same image hash (`502c8012c7a5`) until the layer.yml content changed.

Symptom: `podman image ls` shows a new tag, but `podman run --rm <new tag> cat /path/to/changed/file`
returns the **old** content.

### Known caveat: stale `:latest` under `--cache-from`

ov's Containerfile generator currently emits `FROM <registry>/<builder>:latest`
for parent builder images (not pinned CalVer). When ov invokes
`podman build --cache-from <registry>/<image>`, podman resolves those `FROM`
clauses at parse time and will **pull `:latest` from the remote registry**,
silently clobbering any locally-rebuilt `:latest` tag. For images that rebuild
their builder stages in the same invocation this can lead to the later
stages using the **stale registry builder** instead of the freshly-built one.

**Symptom observed on this project:** `ov update selkies-desktop --build`
fails deep inside selkies' `pixi install && bash build.sh` step with
`error: can't find Rust compiler`. The remote
`ghcr.io/overthinkos/fedora-builder:latest` predates the `build-toolchain`
layer adding `cargo` as an RPM, so its pixi env has no rustc — even though
the current local `build-toolchain/layer.yml` lists `cargo`. The build phase
that rebuilds fedora-builder locally *does* run, but parent-stage FROM
resolution happens before that stage exists, and podman pulls the stale
remote image.

**Workaround until the generator is fixed:**

```bash
ov image build <image> --cache=none      # or equivalently --no-cache at the ov level
```

Both `--cache=none` and `--no-cache` short-circuit `cacheArgs()` in
`ov/build.go:cacheArgs` and do NOT pass `--cache-from` to podman, so
the broken resolution path never fires. `--no-cache` is ov-level only — it
does *not* pass `--no-cache` to podman, it just skips `--cache-from`.

**Proper fix (not yet implemented):** ov's generator should emit pinned
CalVer tags for builder `FROM` clauses (e.g.,
`FROM ghcr.io/overthinkos/fedora-builder:2026.105.0128`) or pass
`--pull=never` to podman so local tags aren't resolved from the remote.
Tracked as a follow-up.

See also: `/ov:generate` for the Containerfile generation path, `/ov:update`
for the `--build` flag that also picks up this caveat.

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + images.yml composition reference
- `/ov:generate` -- Containerfile generation (called internally; stale `:latest` FROM lives there)
- `/ov:inspect` -- Inspect resolved image config before building
- `/ov:list` -- Enumerate images, layers, build targets
- `/ov:merge` -- Post-build layer consolidation (runs inline after each build level)
- `/ov:new` -- Scaffold a new layer directory before adding to `images.yml`
- `/ov:pull` -- Pull prebuilt images; orthogonal to building (use for downstream deploy-mode commands)
- `/ov:validate` -- Validate `images.yml` + layers before building

### Related skills

- `/ov:layer` -- Layer definitions that get built
- `/ov:update` -- `ov update <image> --build` invokes `BuildCmd.Run` and picks up the same `--jobs` cap and stale-`:latest` caveat
- `/ov:vm` -- Building bootc disk images (`ov vm build`)
- `/ov:config` -- Engine configuration
- `/ov:enc` -- Encrypted-volume restart path interacts with the `--build` flow (`ov config mount` short-circuit means builds can restart services without touching the keyring)

## When to Use This Skill

**MUST be invoked** when the task involves building images, pushing to registries, build caches, or layer merging. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Typically first in a lifecycle chain.
Next step: `/ov:deploy` (quadlet setup, tunnels) → `/ov:service` (start and manage).
