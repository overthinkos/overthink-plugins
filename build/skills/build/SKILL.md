---
name: build
description: |
  MUST be invoked before any work involving: building container images, ov box build command, pushing to registries, merging layers, build caches, or Containerfile generation.
---

# ov box build -- Building Container Images

Invoked as `ov box build`. See `/ov-image:image` for the family overview.

## Overview

`ov box build` generates Containerfiles from `box.yml` and layer definitions, then builds images in dependency order using the configured build engine (Docker or Podman). Images at the same dependency level are built in parallel (up to `--jobs` concurrent builds).

**Mode purity**: `ov box build` reads `box.yml` + `build.yml` + `candy.yml` only. `deploy.yml` is never read during build — this is enforced by `LoadConfig` in `ov/config.go`, which calls `LoadConfigRaw` (no `MergeDeployOverlay`) to guarantee OCI labels are baked strictly from authored configuration, never from local deploy-time overrides. See `/ov-internals:go` "Mode purity" for the architectural invariant this protects and the bug it prevents.

**IR-driven emission**: `ov box build` emits Containerfiles via `OCITarget` — the build-mode implementation of the shared `DeployTarget` interface. Internally the flow is: `box.yml` + `candy.yml` → `BuildDeployPlan` (pure compiler) → `InstallPlan` IR → `OCITarget.Emit` → Containerfile text. The same IR backs `PodDeployTarget` and `LocalDeployTarget` used by `ov deploy add`. See `/ov-internals:install-plan` for the IR and `/ov-internals:generate-source` for the Go call graph.

**Three-phase templates**: `build.yml` format and builder definitions split each install operation into `phases.{prepare, install, cleanup}.{container, host}` — three phases × two venues. Build-mode emission reads the `container` cell; local deploys read `host`. A top-level `install_template:` field serves as the `(install, container)` fallback when `phases:` is absent. See `/ov-image:layer` "Service Declaration" for the analogue at the init-system level (`init.<name>.service_schema`).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build all images | `ov box build` | Build all images in dependency order |
| Build specific image | `ov box build <image>` | Build single image for host platform |
| Build and push | `ov box build --push` | Build all platforms and push to registry |
| Build without cache | `ov box build --no-cache` | Disable build cache entirely |
| Merge layers | `ov box merge <image>` | Post-build layer optimization |

## ov box build Commands

```bash
ov box build [image...]                         # Build for local platform
ov box build --push [image...]                  # Build for all platforms and push
ov box build --platform linux/amd64 [image...]  # Specific platform
ov box build --cache registry [image...]         # Registry cache (read+write)
ov box build --cache image [image...]           # Image cache (read-only, default)
ov box build --cache gha [image...]             # GitHub Actions cache
ov box build --no-cache [image...]              # Disable cache entirely
ov box build --jobs N [image...]                # Max concurrent images per DAG level (0=auto from defaults.jobs, else 4)
ov box build --podman-jobs N [image...]         # Max concurrent stages within a single podman build (0=auto: min(NCPU, defaults.podman_jobs_cap))
ov box build <image> --include-disabled         # Build an `enabled: false` image without flipping authored config
```

## `--include-disabled` — operational rebuild of disabled images

Images with `enabled: false` in `box.yml` are excluded from the build /
inspect / validate working set by default. To rebuild ONE such image without
flipping the authored flag (which would commit local-deployment intent into
shared project config), pass `--include-disabled`:

```bash
ov box build immich --include-disabled    # builds the disabled image
git diff --quiet box.yml                  # confirms box.yml is untouched
```

**Scoping.** When you pass positional `<image>` args together with
`--include-disabled`, the override is **scoped to those names**. Other
disabled images stay filtered out — important because widening the
working set globally would surface unrelated dep errors (e.g., a
disabled image with remote layers that haven't been fetched yet).
`ov box build --include-disabled` (no positional args) widens
globally; `ov box build immich --include-disabled` only relaxes the
gate for `immich`.

The flag flows from `BuildCmd.IncludeDisabled` → `ResolveOpts.IncludeDisabled`
+ `ResolveOpts.IncludeDisabledNames` → `ResolveAllImages` (in `Generator`)
→ `ResolveImage`. The `shouldIncludeDisabled(name)` helper centralises the
scoping rule. Sibling commands `ov box inspect <name> --include-disabled`
and `ov box validate --include-disabled` accept the same flag for
diagnostic / validation work on disabled entries. See `/ov-build:inspect`
and `/ov-build:validate`.

## Build Config (build.yml)

Containerfile generation is driven by a single declarative YAML file referenced in `box.yml`:

```yaml
defaults:
  format_config: build.yml    # Unified distro + builder + init config
```

`build.yml` has three top-level sections:

- **`distro:`** — Per-distro bootstrap commands (package manager setup, cache mounts, repo management), optional `base_user:` declaration (what uid-1000 account the upstream base image ships), and package format templates (how `rpm:`, `pac:`, `deb:` sections in candy.yml become `RUN` steps). Each format has `install`, `repos`, `copr`, `modules`, and `options` templates.
- **`builder:`** — Multi-stage builder patterns (pixi, npm, cargo, aur). Each builder has `build_stage` and `copy_stage` templates that generate the appropriate `FROM builder AS ...` and `COPY --from=...` steps.
- **`init:`** — Init system definitions (supervisord, systemd) including detection rules, fragment templates, entrypoint commands, and service management commands. Optional — images can omit this if they don't need an init system.

All sections use Go `text/template` syntax with access to layer config data. Source: `ov/format_config.go` (loader + distro/builder types, including `DistroDef.BaseUser`), `ov/init_config.go` (init type), `ov/format_template.go` (rendering).

### `base_user:` — declaring a pre-existing base-image account

`base_user:` handles upstream base images that ship a pre-existing uid-1000 account (e.g. Ubuntu 24.04's `ubuntu:ubuntu`). Declare it under the distro when the upstream base image ships a uid-1000 account; leave it out otherwise.

```yaml
distro:
  ubuntu:
    inherits: debian
    base_user:
      name: ubuntu
      uid: 1000
      gid: 1000
      home: /home/ubuntu
    # ... bootstrap inherited from debian
```

All four fields (`name`, `uid`, `gid`, `home`) are required when the block is present. Inherited across distro inheritance chains — if the child has no `base_user:` but the parent does, the child inherits it (see `resolveInherits` in `ov/format_config.go`).

Consumed by the `user_policy:` reconciliation in `ov/config.go:ResolveImage` — see `/ov-image:image` "user_policy" for the three-value policy (`auto` / `adopt` / `create`) and the decision matrix.

No `base_user:` currently declared for Fedora, Arch, or Debian (their canonical base images ship no pre-existing uid-1000 account). Add one in your project's `build.yml` override if you're basing on a distro-cloud variant that DOES ship one (e.g. `debian:13-cloud`).

### The `builder:` name in two places

`builder:` appears in both `build.yml` (top-level section) and `box.yml` (per-image map, plus `defaults.builder`). They share the name on purpose — both maps key on the same slot (the build-type name, e.g. `pixi`, `npm`, `cargo`, `aur`):

- `build.yml` `builder.pixi` — **definition**: detection rules, stage template, cache mounts for the pixi builder.
- `box.yml` `builder.pixi` — **selection**: which image (e.g. `fedora-builder`) to use as the pixi builder for this image.

An image's effective builder map resolves as: `image.builder[type]` → `base_image.builder[type]` → `defaults.builder[type]` → `""`. Self-references (a builder image pointing at itself) are filtered automatically.

### Why one file, not three

`build.yml` was unified from three former files (`distro.yml` + `builder.yml` + `init.yml`) because they were always resolved together through the same `format_config:` ref. One file → one loader (`LoadBuildConfigForImage`) → three in-memory configs (`DistroConfig` / `BuilderConfig` / `InitConfig`) — the internal split is preserved, only the YAML surface is unified. The `init:` section is optional (absent = no init system); `distro:` and `builder:` are required.

## Containerfile Generation

`ov box build` runs `ov box generate` internally. You can also run it standalone to inspect generated Containerfiles:

```bash
ov box generate                          # Write .build/ (Containerfiles)
ov box generate --tag v1.0.0             # Override CalVer tag
cat .build/my-image/Containerfile    # Inspect generated output
```

## Build Flow

1. Run `ov box generate` internally (produces Containerfiles in `.build/`)
2. Resolve runtime config to get build engine (`engine.build`)
3. Resolve image build order (dependency ordering, grouped by level)
4. Filter to requested images (and their base dependencies)
5. For each level: build images in parallel (up to `--jobs` concurrent, default 4)
6. After all builds: `ov box merge --all` (if `merge.auto` enabled, skipped for `--push`)

## Parallelism: `--jobs` vs `--podman-jobs`

ov exposes **two** parallelism knobs with distinct meanings:

| Flag | Env var | `defaults:` key | What it controls |
|---|---|---|---|
| `--jobs N` | `OV_BUILD_JOBS` | `jobs` | **Outer** concurrency: how many ov-level images to build in parallel within a DAG level (e.g., when `ov box build` rebuilds the whole graph). |
| `--podman-jobs N` | `OV_PODMAN_JOBS` | `podman_jobs` (+ `podman_jobs_cap`) | **Inner** concurrency: passed to `podman build --jobs N`, controls how many stages of a *single* multi-stage build run concurrently. |

Both knobs are **config-driven**: precedence is **CLI flag → env → `defaults:`
in `overthink.yml` → built-in fallback** (4). The inner auto value (when
`--podman-jobs` / `OV_PODMAN_JOBS` / `defaults.podman_jobs` are all unset) is
CPU-proportional, capped at `defaults.podman_jobs_cap`: `min(NCPU, cap)`. The
cap is the operative ceiling; the repo ships `podman_jobs_cap: 8`. (A
high-concurrency `--cache-from` SIGABRT race in podman ≤ 5.7.x originally
motivated a hard cap of 4 — see `CHANGELOG.md` for that history and the 20-run
race gate required before raising the cap on a new podman version.)

```yaml
# overthink.yml — defaults: block
defaults:
  jobs: 4              # outer: images per DAG level
  podman_jobs: 0       # inner: 0 = auto = min(NCPU, podman_jobs_cap)
  podman_jobs_cap: 8   # ceiling for the auto calc
```

```bash
ov box build <image> --podman-jobs 16             # fully parallel stages (override)
OV_PODMAN_JOBS=8 ov update <image> --build    # via env
ov box build <image> --podman-jobs 1              # fully serialised, worst-case debugging
```

Source: `ov/build.go:resolvePodmanJobs(override, cap)` + `podmanJobsCapFallback`
/ `jobsFallback`; config fill in `BuildCmd.Run`. Covered by
`ov/build_jobs_test.go`. The outer `--jobs` knob and the inner `--podman-jobs`
are separate fields so the two semantics don't get conflated.

## Build-context excludes (`defaults.context_ignore`)

`ov box generate` writes BOTH `.containerignore` and `.dockerignore` at the
project root from a single source: a built-in baseline (`.git`, `bin`, `ov`,
`*.md`, plus editor/python/node cache-bust globs) **plus** every entry in
`defaults.context_ignore`. Both files are generated artifacts (gitignored) — do
not hand-edit them; add excludes to `defaults.context_ignore` instead.

```yaml
# overthink.yml — defaults: block
defaults:
  context_ignore:        # heavy dirs never COPYed by any Containerfile
    - image
    - .eval
    - output
    - pkg
    - tests
```

Why it matters: the build context tar is streamed to the engine on **every**
build regardless of cache state, so excluding large never-COPYed directories is
the dominant warm-rebuild win. podman reads `.containerignore`; docker reads
`.dockerignore` — emitting both keeps the two engines in lockstep. Only add a
directory you've confirmed no Containerfile COPY/ADDs from (generated
Containerfiles COPY only from `candy/`, `templates/`, `.build/`). Source:
`ov/generate.go:writeContextIgnore` + `baselineContextIgnore`.

## Image-tag retention (`defaults.keep_images`)

After `ov box build` (push runs excluded), ov prunes old CalVer tags per image
down to `defaults.keep_images` — keeping the newest N builds per
`org.overthinkos.image` group, ordered by the `org.overthinkos.version` label
(the content-derived EffectiveVersion) as the PRIMARY key, with the `:YYYY.DDD.HHMM`
build TAG as the tiebreaker. The tag tiebreak is load-bearing: the label is
content-stable, so many builds of an unchanged image share one label-CalVer and
the tag is what distinguishes (and retains) the newest BUILDS. Images referenced
by a container (`podman ps -a`) are skipped, and `rmi` runs without `-f` as a
backstop, so a running deploy's image is never removed.

```yaml
# overthink.yml — defaults:
defaults:
  keep_images: 3   # newest CalVer tags to keep per image; 0 (or absent) disables
```

This stops the iterative-build tag accumulation that otherwise reclaims
nothing. Run it on demand (and clear a backlog) with `ov clean` — see
`/ov-core:clean` for the full retention surface (images + eval runs + makepkg).

## Build Cache

| Mode | Backend | Use Case |
|------|---------|----------|
| `image` | `<registry>/<image>` | Read-only cache from registry image (default) |
| `registry` | `<registry>/cache:<image>` | Production CI/CD (read + write) |
| `gha` | GitHub Actions cache | CI builds on GitHub Actions |
| `none` | No cache | Same as `--no-cache` |

```bash
ov box build --cache registry my-image        # Read+write registry cache
ov box build --cache image my-image          # Read-only from registry image
OV_BUILD_CACHE=registry ov box build         # Via environment variable
```

The default cache mode is also config-driven: precedence is `--cache` →
`OV_BUILD_CACHE` → `defaults.cache` in `overthink.yml` → auto (`image` for local
builds, `registry` for `--push`). Set `defaults.cache: image` to make the
read-only registry cache the project default without per-invocation flags.

## CalVer-only tagging (no `:latest`)

`ov box build` tags every image with exactly one tag — its CalVer
(e.g. `ghcr.io/overthinkos/fedora-supervisord:2026.114.1022`). ov does
**not** emit a `:latest` tag, ever. Short-name resolution (in
`ov/local_image.go`) picks the newest CalVer for a given short name
via the `org.overthinkos.image=<short>` + `org.overthinkos.version=<calver>`
OCI labels. The CLI accepts an explicit `--tag <calver>` for pinning;
an empty `--tag` resolves to newest-local automatically.

Rationale:
1. **No stale-`:latest`-under-`--cache-from` bug.** The documented
   caveat where podman's `--cache-from` silently pulls a stale
   `:latest` from the remote registry is unreachable when ov never
   emits `:latest` in the first place.
2. **No "ambiguous short name" errors under multiple tags.** Previously
   the resolver listed every `:latest` + `:<calver>` tag of the same
   image as competing candidates; the CalVer sort picks the newest
   one deterministically.
3. **One-tag-per-build keeps registry + local storage lean.** A float
   tag is a permanent nameable handle you have to garbage-collect;
   a CalVer tag is self-describing and rotates naturally.

Any stray `:latest` tag in local storage is never refreshed by a new
build; prune it with `podman image prune` if you want it gone.

## Cache Efficiency

### Core invariant — cache hits fully when nothing has changed

**Running `ov box build <image>` with no source changes completes in seconds — every RUN/COPY/LABEL step hits the cache.** This is the invariant the rest of this section builds on. Cache-miss only happens when something in the build input genuinely changes; a rebuild you triggered "just to be sure" is free.

Concretely: the cache is keyed by `(parent-image-SHA, instruction-text, COPY-source-content)`. The parent-image SHA resolves from the FROM reference at build time — a fresh CalVer tag that still points at the same image SHA keeps the subsequent RUN/COPY steps cached. Only when the underlying image SHA changes (because that upstream was rebuilt with changed content) does cache-miss cascade to the downstream steps.

### What legitimately invalidates cache

Three kinds of source changes are real cache invalidators — if you see a long rebuild, one of these is the cause:

1. **Layer source file content changed.** Editing a file under `candy/<name>/` — the canonical case is `candy/ov/bin/ov` being rewritten by `task build:ov` after a Go source edit — changes the scratch stage's content hash, which invalidates `COPY --from=<layer>` and everything downstream that depends on it.
2. **Package list / task text changed.** Adding/removing an rpm/deb/pac entry or editing a `cmd:` body changes the RUN instruction text emitted for that layer, invalidating cache from that RUN onward.
3. **Upstream image content changed.** If a base image (external like `fedora` or internal like `fedora-supervisord`) has different content from the last cached build, the FROM step resolves to a new SHA and downstream RUN/COPY steps all cache-miss. This cascades through the dependency graph — rebuilding `fedora-supervisord` forces its children to re-run from the `FROM fedora-supervisord` step.

### What does NOT invalidate cache

- **CalVer tag shifts alone.** The Containerfile emits `ARG BASE_IMAGE=<registry>/<name>:<calver>` with a fresh CalVer on every generate. That ARG default appears in the Containerfile text but is not part of the cache key for subsequent RUN/COPY steps — podman/buildah resolves the FROM to an image SHA first, and caches downstream steps off that SHA. If the SHA is unchanged, cache hits. The cache cost comes from content changes in the layer itself, not the tag.
- **`eval:` edits.** LABEL directives are emitted last in every final stage (after the last USER). A `eval:` edit — the most common layer mutation — only re-runs the final LABEL block (~2 seconds on a 138-step stack like `immich-ml`). See `/ov-internals:generate-source` for the rationale.
- **Re-running `ov box build` without source changes.** Fully cached; seconds to complete.

### `write:` vs `copy:` — cache granularity

- `write:` tasks use `stageInlineContent` (content-addressed staging under `.build/<image>/_inline/<layer>/<sha256>/`). Editing a `write: content:` block changes only that single COPY layer's cache key — siblings in the same layer keep their cache.
- `copy:` tasks reference files from the layer directory. Editing any file under `candy/<name>/` changes the WHOLE scratch stage's content hash (since `COPY candy/<name>/ /` is one instruction), invalidating every downstream `COPY --from=<layer>` step.

### Rule of thumb for rebuild cost

| Edit | Cost |
|------|------|
| `ov box build` with zero source changes | Seconds — every step cache-hits |
| A `eval:` / label entry | ~2 sec (LABEL re-emit only) |
| A `write:` task's content | Just that single content-addressed COPY layer |
| A `copy:` source file's content | Rebuild from that layer's COPY onward + downstream |
| A `cmd:` / `download:` task body | Rebuild from that RUN onward + downstream |
| A package added/removed in `rpm:`/`deb:`/`pac:` | Rebuild from the install RUN onward + downstream |
| `task build:ov` → new `candy/ov/bin/ov` | Rebuild the `ov` layer + every image that includes it |
| An upstream image got content-changed and rebuilt | Rebuild from the FROM step onward in every descendant |

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
ov box merge <image> --dry-run         # Preview what would be merged
ov box merge <image>                   # Merge small layers
ov box merge --all                     # Merge all images with merge.auto enabled
ov box merge <image> --max-mb 512      # Custom per-layer threshold
ov box merge <image> --max-total-mb 4096  # Custom total image size limit
```

Configure in box.yml:

```yaml
defaults:
  merge:
    auto: true        # Auto-merge after builds
    max_mb: 128       # Max merged layer size (MB)
    max_total_mb: 0   # Max total image size for merge (0 = no limit)
```

CLI flags `--max-mb` and `--max-total-mb` override `box.yml`. `auto` is only used by `ov box merge --all` to select which images to merge; `ov box merge <image>` always merges regardless. `max_total_mb` controls whether large images skip merging entirely (the merge process decompresses layers in memory). Set to `0` to disable (default), or a positive value like `2048` to cap on low-memory CI runners.

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

For filtered builds (`ov box build <image>`), only the built images are merged. For full builds (`ov box build`), merge runs after each dependency level completes.

## Engine Configuration

```bash
ov settings set engine.build docker   # or podman
ov settings set engine.run docker     # or podman
```

## Host Bootstrap (First Time)

Requires: `go-task`, `go`, `docker` (or `podman`). On Arch the recommended install is `cd pkg/arch && makepkg -si` — the bundled `overthink-git` PKGBUILD is LOCAL-ONLY (it is NOT published to the AUR, so there is no `yay -S overthink-git`); it pulls every dep, and the bundled pacman post-install hook enables docker/tailscaled/virtqemud automatically. (`makepkg -si` resolves the AUR-only mandatory deps via an AUR helper, or pre-install them — see the PKGBUILD's `makedepends`/AUR notes.) On other distros, run `task build:ov` from the checkout to compile and install `ov` to `~/.local/bin/ov`.

```bash
task build:ov        # Build + install ov; on Arch delegates to makepkg -si, elsewhere installs portable to ~/.local/bin
task setup:builder   # Create multi-platform buildx builder
ov box build       # Generate + build + merge all images
```

## Common Workflows

### Build a Single Image

```bash
ov box build my-app
```

### Rebuild After Layer Changes

```bash
ov box build my-app
# Only changed layers are rebuilt (Docker layer cache)
```

### Push to Registry

```bash
# Authenticate first
docker login ghcr.io
# or: podman login ghcr.io

ov box build --push
```

## Troubleshooting

### "ov not found"

On Arch run `cd pkg/arch && makepkg -si` to install the bundled `overthink-git` package system-wide (it is LOCAL-ONLY — NOT on the AUR — so `yay -S overthink-git` will not find it). Elsewhere run `task build:ov` from the checkout — `task` itself is required (install via your distro package manager or download from go-task/task releases).

### Build Fails with Missing Base

Build base images first. `ov box build` handles dependency ordering automatically, but if building a single image, its base must already exist.

### Cache Miss

First build on a new machine won't have cache. Use `--cache registry` to pull from registry cache if available.

### RPM Conflict: ffmpeg-free vs negativo17

If a build fails with `conflicting requests` involving `libavcodec-free` vs `libavcodec` (epoch 1), the layer is trying to install `ffmpeg-free` (Fedora) in an image that has negativo17's `ffmpeg-libs` (via cuda layer). Fix: change `ffmpeg-free` to `ffmpeg` in the layer's `rpm.packages` and add the `fedora-multimedia` repo from negativo17. See the `immich` layer for the correct pattern.

### YAML Unmarshal Error on candy.yml

If you see `cannot unmarshal !!str ... into int` or similar YAML parsing errors on layer fields, the installed `ov` binary is likely stale. Rebuild with `task build:install` or `cp bin/ov ~/.local/bin/ov`. Verify with `ov box validate`.

### Stale `ov` binary produces stale Containerfiles

Beyond the YAML-unmarshal symptom above, a stale `ov` binary can produce *syntactically valid but outdated* Containerfile output — e.g. emitting an old broken form of a template that HEAD's source has already fixed. Symptom: build fails on a step whose generated shell clearly doesn't match the source you see in `git grep`. Quick diagnostic: `ls -la $(which ov)` vs. `git log -1 ov/generate.go` — if the binary predates the fix, rebuild:

```bash
task build:ov        # rebuild + pacman-reinstall on Arch (overthink-git package)
```

Common on Arch where `overthink-git` is pacman-installed and HEAD moves faster than rebuilds. If you find yourself rebuilding `ov` to chase a bug and the symptom persists, the binary path on $PATH may not be the one `task build:ov` updated — confirm with `which ov`.

### Buildah cache-mount corruption (pixi tzdata et al.)

Cache mounts declared via `--mount=type=cache,dst=<path>,uid=<u>,gid=<g>` (used for pixi, npm, cargo, rattler, dnf) persist across builds and are **not** evicted by `ov box build --no-cache` (which only suppresses `--cache-from`; see above). A disk-full event mid-download can leave a partial package in the cache, and every subsequent build re-hits the same broken file:

```
error: × failed to link tzdata-2025c-hc9c84f9_1.conda
  ├─▶ failed to copy file ... zoneinfo/<Region>/<City>: No such file or directory
```

Options, in order of least to most disruptive:

1. **Just retry the build.** Often the cache clears itself once the partial package finishes downloading or gets superseded by a checksum mismatch.
2. **Bump a content-hash on the builder layer** (e.g. `echo "" >> candy/pixi/candy.yml`) to evict the cache mount associated with that build-stage's hash, then revert the cosmetic change. Same workaround as the scratch-stage cache issue above.
3. **`podman image prune -af` is NOT the right hammer** — see next note.

### `podman image prune -af` caveat — removes tagged-but-idle images

`podman image prune -a -f` will delete tagged images that are **not referenced by a running container**. That means the image you just built via `ov box build` — and haven't started as a container yet — is eligible for prune. Observed sequence: `ov box build` → `podman image prune -af` → `sudo podman load` (manual rootful refresh) fails with `image not known`. Protect the fresh build by starting a container or use `prune` with explicit filters instead.

### `--no-cache` does not invalidate intermediate scratch-stage caches

`ov box build --no-cache <image>` and `ov box build --cache none <image>` reliably disable the
cache for the **final image stage**, but in observed behavior they do **not** propagate
to intermediate scratch stages produced by `COPY candy/<x>/ /` instructions
(`[15/25] STEP 2/2: COPY candy/labwc/ /` style). Editing a single file inside a layer
directory and rebuilding with `--no-cache` may still pull the labwc scratch stage from
cache, leaving the new file content out of the rebuilt image.

**Workaround:** force a content-hash bump on the layer's `candy.yml`. The simplest is
adding (or removing) a trailing comment line:

```bash
echo "" >> candy/labwc/candy.yml          # bump content hash
ov box build selkies-desktop                   # now invalidates the labwc scratch stage
git checkout -- candy/labwc/candy.yml     # revert the cosmetic change
```

This was discovered while shipping commit `febb9bd` (labwc autostart race fix): the
edit to `candy/labwc/autostart` did not propagate to the rebuilt image until
`candy/labwc/candy.yml` itself was touched. Two consecutive `--no-cache` rebuilds
produced the same image hash (`502c8012c7a5`) until the candy.yml content changed.

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
the current local `build-toolchain/candy.yml` lists `cargo`. The build phase
that rebuilds fedora-builder locally *does* run, but parent-stage FROM
resolution happens before that stage exists, and podman pulls the stale
remote image.

**Workaround until the generator is fixed:**

```bash
ov box build <image> --cache=none      # or equivalently --no-cache at the ov level
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

See also: `/ov-build:generate` for the Containerfile generation path, `/ov-core:ov-update`
for the `--build` flag that also picks up this caveat.

## Project directory override

`ov box build` (like every build-mode command) resolves `box.yml` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>` — honoured before Kong dispatch. See `/ov-image:image` "Project directory resolution" for the canonical reference and the `ov mcp serve` use case. Typical use: building from an `ov mcp serve` MCP tool where the container cwd doesn't hold the project.

## Cross-References

### `ov box` family siblings

- `/ov-image:image` -- Family overview + box.yml composition reference
- `/ov-build:generate` -- Containerfile generation (called internally; stale `:latest` FROM lives there)
- `/ov-build:inspect` -- Inspect resolved image config before building
- `/ov-build:list` -- Enumerate images, layers, build targets
- `/ov-build:merge` -- Post-build layer consolidation (runs inline after each build level)
- `/ov-build:new` -- Scaffold a new layer directory before adding to `box.yml`
- `/ov-build:pull` -- Pull prebuilt images; orthogonal to building (use for downstream deploy-mode commands)
- `/ov-build:validate` -- Validate `box.yml` + layers before building

### Related skills

- `/ov-image:layer` -- Layer definitions that get built
- `/ov-eval:eval` -- Tests are embedded as `org.overthinkos.eval` OCI label at build time; LABEL-at-end optimization (see Cache Efficiency above) makes test edits cheap.
- `/ov-core:ov-update` -- `ov update <image> --build` invokes `BuildCmd.Run` and picks up the same `--jobs` cap and stale-`:latest` caveat
- `/ov-vm:vm` -- Building bootc disk images (`ov vm build`)
- `/ov-core:ov-config` -- Engine configuration
- `/ov-automation:enc` -- Encrypted-volume restart path interacts with the `--build` flow (`ov config mount` short-circuit means builds can restart services without touching the keyring)

## When to Use This Skill

**MUST be invoked** when the task involves building images, pushing to registries, build caches, or layer merging. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Typically first in a lifecycle chain.
Next step: `/ov-core:deploy` (quadlet setup, tunnels) → `/ov-core:service` (start and manage).

## Related skills

- `/ov-internals:capabilities` — OCI labels emitted during the build stage; `CapabilityLabelMap` completeness check

## Live-deploy verification is mandatory (see `/ov-eval:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-internals:disposable`). Use `ov update <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vm.yml.

**After committing the source-level fix, `ov update` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
