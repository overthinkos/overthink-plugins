---
name: generate
description: |
  Containerfile generation from image.yml and layers.
  MUST be invoked before any work involving: ov image generate command, Containerfile generation, .build/ directory contents, the task-verb emission pipeline, or understanding generated output.
---

# ov image generate -- Containerfile Generation

Invoked as `ov image generate`. See `/ov:image` for the family overview.

## Overview

Parses `image.yml`, scans `layers/`, resolves the dependency graph, and emits all build artifacts into the `.build/` directory. Called internally by `ov image build` but can be run standalone to inspect generated output before a build.

The generator is **config-driven** — distro format templates, builder stage templates, and init system fragments all come from a single `build.yml` (three top-level sections: `distro:`, `builder:`, `init:`) — and **declarative per-task** for install logic — each task verb (see `/ov:layer`) has a dedicated emitter that writes the right Containerfile directive.

**IR-driven since 2026-04**: `ov image generate` now drives emission through the shared `DeployTarget` interface. `image.yml` + `layer.yml` compile into an `InstallPlan` IR (one per layer); `OCITarget.Emit` walks the IR and writes Containerfile text. The same IR backs `ContainerDeployTarget` (overlay-Containerfile synthesis when `add_layers:` is set) and `HostDeployTarget` (host-target execution). See `/ov-dev:install-plan` for the type catalog and `/ov-dev:generate` for the Go-level call graph.

**Three-phase templates**: `build.yml` format (`formats.<fmt>`) and builder (`builders.<name>`) definitions carry a new `phases: { prepare, install, cleanup }.{ container, host }` structure. The generator reads `phases.install.container` when set and falls back to the legacy top-level `install_template:` otherwise. The `host:` cell is consumed only by `HostDeployTarget` (`ov deploy add host`) — the generator ignores it. Template migration is layer-by-layer; both paths coexist during the transition.

## Quick Reference

| Action | Command | Description |
|---|---|---|
| Generate all | `ov image generate` | Generate Containerfiles for all enabled images |
| With tag | `ov image generate --tag TAG` | Override the image tag |

`ov image generate` takes **no positional image argument** — it always writes the full `.build/` tree for every enabled image in `image.yml`. To inspect a single image's output, run `ov image generate` (fast — it reuses scratch-stage caches) and then `cat .build/<image>/Containerfile`. Filtering to one image happens implicitly via `ov image build <image>`, which invokes generate internally and then builds only the requested image + its dependencies.

```bash
# Generate all Containerfiles
ov image generate

# Generate with a custom tag
ov image generate --tag v1.2.3

# Inspect generated output
cat .build/fedora/Containerfile
```

## Generated Output

| Path | Description |
|---|---|
| `.build/<image>/Containerfile` | The generated Containerfile — unconditional RUN / COPY / ENV / LABEL directives |
| `.build/<image>/_inline/<layer>/<sha256>` | Inline-content bytes from `write:` tasks (content-addressed, idempotent) |
| `.build/<image>/traefik-routes.yml` | Traefik dynamic config (only for images with `route:` layers) |
| `.build/<image>/<fragment_dir>/*.conf` | Init-system service configs (e.g. `supervisor/`, `systemd/`) |
| `.build/_layers/<name>` | Symlinks to remote layer directories |

All generated files start with `# <path> (generated -- do not edit)`.

## Per-layer emission pipeline

For each layer, `writeLayerSteps` runs this sequence:

```
1. Comment header:    # Layer: <name>
2. ENV from `vars:` + ARG TARGETARCH + ENV ARCH=${TARGETARCH} (once per layer)
3. Package install (rpm/deb/pac/aur) — always USER root
4. tasks: iterated in author order:
   a. Resolve ${VAR} in non-verbatim fields
   b. Determine user: (default root)
   c. Emit USER <value> if different from running USER
   d. Dispatch to the verb-specific emitter
   e. Adjacent same-verb same-user tasks coalesce into one directive
      (mkdir, link, setcap) — see /ov:layer execution-order section
   f. Parent-dir auto-insertion for copy/write when no earlier mkdir covers
5. Builders (pixi/npm/cargo/aur) — placement is end-of-layer unless an
   explicit `- build: all` task appears in tasks:
6. Reset to USER root (unless last layer and no further root steps follow)
```

### Per-verb emitters (single Go file: `ov/tasks.go`)

| Verb | Emitter | Containerfile output |
|---|---|---|
| `mkdir` | `emitMkdirBatch` | `RUN mkdir -p p1 p2 … [ && chmod <mode> p1 … ]` (one RUN per batch; grouped by mode) |
| `copy` | `emitCopy` | `COPY --from=<layer-stage> --chmod=<mode> [--chown=<uid>:<gid>] <src> <dest>` — **no RUN** |
| `write` | `emitWrite` | `COPY --from=<layer-stage> --chmod=<mode> [--chown=] .build/<image>/_inline/<layer>/<sha256> <dest>` — **no RUN, no shell heredoc** |
| `link` | `emitLinkBatch` | `RUN ln -sf t1 l1 && ln -sf t2 l2 …` (one RUN per batch) |
| `download` | `emitDownload` | `RUN --mount=type=cache,dst=/tmp/downloads bash -c 'export BUILD_ARCH=$(uname -m); curl -fsSL <url> \| <extractor>'` (one RUN per download; `export ...;` termination is required so bash expands `${BUILD_ARCH}` in the URL) |
| `setcap` | `emitSetcapBatch` | `RUN setcap -r … && setcap caps path …` (strip + set chained) |
| `cmd` | `emitCmd` | `RUN --mount=type=bind,from=<layer-stage>,source=/,target=/ctx [--mount=type=cache,…] bash -c $'BUILD_ARCH=$(uname -m)\nset -e\n<command>'` — ANSI-C `$'...'` quoting keeps the multi-line body on one physical line (podman's Dockerfile parser splits at unescaped newlines) |
| `build` | handled inline in `writeLayerSteps` | Existing pixi/npm/cargo/aur multi-stage + inline blocks |

## Cache-mount inheritance

Cache mounts come from `build.yml` — `distro:` section (format caches) and `builder:` section (builder caches). `ov tasks.go` picks the right set based on task context:

| Task context | Cache mount(s) |
|---|---|
| `cmd:` as root | Distro format caches (`/var/cache/libdnf5` / `/var/cache/apt` / `/var/cache/pacman/pkg`) + `/ctx` bind mount |
| `cmd:` as non-root | `/tmp/npm-cache` (UID-scoped) + `/ctx` bind mount |
| `download:` | `/tmp/downloads` (shared across layers) |
| Package install | Distro format caches from `build.yml` `distro:` section |
| pixi builder | `/tmp/pixi-cache` + `/tmp/rattler-cache` (UID-scoped) |
| npm builder | `/tmp/npm-cache` (UID-scoped) |

The `/ctx` bind mount exposes the layer's own directory tree to `cmd:` tasks — so you can still reference `/ctx/<file>` inside an escape-hatch shell block for one-off file access (though `copy:` / `write:` are strongly preferred).

## USER emission

- `user: root` → `USER 0`
- `user: ${USER}` → `USER <numeric UID>` (matches the pre-refactor convention; avoids `/etc/passwd` dependency at the switch point)
- `user: <uid>:<gid>` → `USER <uid>:<gid>` (e.g. `1010:1010`)
- `user: <name>` → `USER <name>` (literal; requires user to exist — create via earlier `cmd: useradd` task)

`COPY --chown=` uses numeric `<UID>:<GID>` for `${USER}` (BuildKit-safe), name-pairs for literal users.

## writeBootstrap — adopt vs create (2026-04)

`writeBootstrap` emits the user-creation section of the base-image Containerfile and branches on `ResolvedImage.UserAdopted`:

**Adopt mode** (`UserAdopted = true`) — the base image already ships the declared user; no `useradd` is needed. Emits:

```dockerfile
# User ubuntu (uid=1000) adopted from base image (declared in build.yml distro.base_user) — no useradd needed

WORKDIR /home/ubuntu
USER 1000
```

**Create mode** (`UserAdopted = false`) — classic idempotent useradd:

```dockerfile
RUN if ! getent passwd 1000 >/dev/null 2>&1; then \
      (getent group 1000 >/dev/null 2>&1 || groupadd -g 1000 user) && \
      useradd -m -u 1000 -g 1000 -s /bin/bash user; \
    fi

WORKDIR /home/user
USER 1000
```

The pivot is `ResolvedImage.UserAdopted`, set by the `user_policy:` reconciliation in `ov/config.go:ResolveImage`. See `/ov:image` "user_policy" for the policy semantics, `/ov:build` "base_user:" for the declaration, and `/ov-images:ubuntu` for the canonical adopt-mode worked example.

Neither branch does destructive metadata mutation (no `usermod -l` rename). Fedora/Arch/Debian always hit the create branch (no `base_user:` declared); Ubuntu under `user_policy: auto` hits the adopt branch.

## Tag-section install emission

Distro-version tag sections like `debian:13:` and `ubuntu:24.04:` are resolved via first-match-wins on the image's `distro:` priority list (e.g. `["ubuntu:24.04", "ubuntu", "debian"]`). Each matched tag section uses the primary format's full install template — so a tag section can carry `repos:`, `keys:`, `options:`, and `packages:`, not just packages alone. See `ov/layers.go:TagPkgConfig.Raw` for the map that captures full tag-section YAML, and `/ov:layer` for authoring reference.

## ARCH / TARGETARCH emission

Every layer section begins with:

```dockerfile
ARG TARGETARCH
ENV ARCH=${TARGETARCH}
```

`${ARCH}` is then resolvable by BuildKit's ENV substitution in subsequent `COPY` paths, `ENV` values, and inside `RUN` shell. `${BUILD_ARCH}` is uname-style and auto-injected as a shell-local variable at the top of each `cmd:` / `download:` RUN (`BUILD_ARCH=$(uname -m)`), so multi-arch URL templates using either form work without author ceremony.

## Inline-content staging (`write:` verb)

`write: content:` is written to `.build/<image>/_inline/<layer>/<sha256>` at generate time, where `<sha256>` is the SHA-256 of the content. This means:

- **Idempotent:** rewriting the same content is a no-op
- **Content-addressed:** editing the content changes the hash, which changes the COPY source path, which invalidates only that single COPY layer's cache
- **No shell heredoc:** the COPY delivers the bytes directly; no `$`, backticks, or `EOF` markers need escaping

The Containerfile references the file by its relative path: `COPY --from=<layer-stage> --chmod=<mode> .build/<image>/_inline/<layer>/<sha256> <dest>`.

## Behavior

- Generation is idempotent — safe to run repeatedly.
- `.build/` is disposable and gitignored; `ov image generate` will recreate it from scratch.
- Layer dependencies resolve transitively and topologically; circular `depends:` is a validation error (surfaced by `/ov:validate`).
- Pixi manylinux fix is injected into `pixi.toml` files during the pixi builder stage.
- Multi-stage builds use builder images declared in `build.yml` `builder:` section (`pixi-builder`, `npm-builder`, `archlinux-builder` for AUR, etc.).
- Stale `.build/<image>/` directories (from removed or renamed images) are cleaned at the start of each generation.

### LABEL placement (cache efficiency)

All `org.overthinkos.*` LABEL directives are emitted at the **end** of
the final stage, after the last `USER` directive. This means a test or
label edit only re-runs the LABEL steps themselves (metadata-only, ~2
sec) instead of invalidating the buildkit cache for every upstream
RUN/COPY. Particularly important for test authoring: `tests:` edits on
a 138-step stack like `immich-ml` used to cost minutes per iteration;
they now cost seconds. See `/ov-dev:generate` "LABEL Placement" for the
rationale and `/ov:test` for author-facing workflow implications.

## Bootc-specific generator behaviour

Three emission rules matter specifically for bootc images. The canonical worked example is `/ov-images:selkies-desktop-bootc`.

### 1. `initHasFragments` pre-scan gates empty init stages

Each init system defined in `build.yml` (currently `supervisord` and `systemd`) emits a `FROM scratch AS <stage_name>` scratch stage that layers COPY fragments into, plus an `assembly_template` RUN that bind-mounts from that scratch stage. `ov/generate.go` pre-scans the layer chain for each init system to check whether any layer contributes fragments (per-entry rendered from the unified `service:` list via `ServiceSchema.ServiceTemplate` / `ServiceSchema.SupportsPackaged`, plus relay configs, plus systemd `.service` files). If none, **both** the scratch stage and the assembly_template RUN are suppressed. Without this, a bootc image with only packaged-unit entries (`use_packaged:`, no rendered body) would emit an empty `FROM scratch AS systemd-services` + a RUN that bind-mounts from it — which fails at build time. The `system_enable_template` and `post_assembly_template` for that init still run — they're independent of the scratch stage.

### 2. `anyRepoHasURL` helper → prepend `dnf5-plugins`

The RPM install_template prepends `dnf install -y dnf5-plugins` whenever any layer `rpm.repos:` entry declares a `url:` (checked via the `anyRepoHasURL` template helper in `ov/format_template.go`). Required because `quay.io/fedora/fedora-bootc:43` strips `dnf5-plugins` (which provides the `config-manager` subcommand) from the default install. Without the prepend, `dnf5 config-manager addrepo --from-repofile=…` — emitted for every URL-based repo — fails with `Unknown argument "config-manager" for command "dnf5"`. `/ov-layers:ffmpeg` is the canonical URL-repo consumer (adds negativo17's `fedora-multimedia.repo`).

### 3. `export BUILD_ARCH=…;` (not prefix assignment) in `download:` tasks

The `download:` task emits `export BUILD_ARCH=$(uname -m); curl -fsSL "…${BUILD_ARCH}…"` with an explicit semicolon-separator `export`, not the prefix form `BUILD_ARCH=$(uname -m) curl ...`. Bash prefix assignments set the variable in the spawned command's environment **after** the shell has already expanded `${BUILD_ARCH}` in the command's arguments — the expansion sees an unset variable, the URL resolves with an empty arch string, and the download 404s. Source: `ov/tasks.go:envPrefix` with the documenting comment. Layers that use `${BUILD_ARCH}` in a `download:` URL: `/ov-layers:pixi`, `/ov-layers:typst`, `/ov-layers:ujust`, `/ov-layers:yay`, `/ov-layers:vectorchord`, `/ov-layers:sherpa-onnx`.

## Project directory override

`ov image generate` resolves `image.yml` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. See `/ov:image` "Project directory resolution".

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + image.yml composition reference
- `/ov:build` -- Building images (calls generate internally)
- `/ov:inspect` -- Inspect generated output for a specific image
- `/ov:list` -- Enumerate targets before generation
- `/ov:merge` -- Post-build layer consolidation
- `/ov:new` -- Scaffold a new layer to generate into
- `/ov:pull` -- Pull prebuilt images (orthogonal to generate)
- `/ov:validate` -- Validation rules for images and layers (including per-verb task rules)

### Related skills

- `/ov:layer` — **Canonical task verb catalog, `vars:` substitution, YAML anchors, execution order.** Read this first for authoring questions.
- `/ov:test` — test-authoring workflow; `tests:` blocks are embedded via `writeJSONLabel` and benefit directly from LABELs-at-end cache efficiency.
- `/ov-dev:generate` — Deep dive on Containerfile emission internals, `Task` struct, per-verb emitters, `stageInlineContent`, `shellSingleQuote` + `shellAnsiQuote` helpers, LABEL-placement rationale.
- `/ov-dev:go` — Source-code map: `ov/tasks.go` (~430 lines), `ov/generate.go:writeLayerSteps` + `writeLabels`, `ov/layers.go` struct definitions.
- `/ov-images:selkies-desktop-bootc` — canonical worked example exercising all three bootc-specific emission rules above.
- `/ov-layers:ffmpeg` — canonical URL-repo consumer (triggers the `dnf5-plugins` prepend).
- `/ov-layers:bootc-config` — bootc boot wiring that depends on the empty-init-stage fix (only `use_packaged:` entries, no custom-exec rendered bodies).
