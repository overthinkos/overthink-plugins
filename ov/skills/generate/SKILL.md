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

The generator is **config-driven** (distro format templates from `distro.yml`, builder stage templates from `builder.yml`, init system fragments from `init.yml`) and **declarative per-task** for install logic — each task verb (see `/ov:layer`) has a dedicated emitter that writes the right Containerfile directive.

## Quick Reference

| Action | Command | Description |
|---|---|---|
| Generate all | `ov image generate` | Generate Containerfiles for all enabled images |
| With tag | `ov image generate --tag TAG` | Override the image tag |
| Single image | `ov image generate <image>` | Generate for a specific image (not all flavours support this — see below) |

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
| `download` | `emitDownload` | `RUN --mount=type=cache,dst=/tmp/downloads bash -c 'BUILD_ARCH=$(uname -m) curl -fsSL <url> \| <extractor>'` (one RUN per download) |
| `setcap` | `emitSetcapBatch` | `RUN setcap -r … && setcap caps path …` (strip + set chained) |
| `cmd` | `emitCmd` | `RUN --mount=type=bind,from=<layer-stage>,source=/,target=/ctx [--mount=type=cache,…] bash -c 'BUILD_ARCH=$(uname -m)\nset -e\n<command>'` (one RUN per cmd) |
| `build` | handled inline in `writeLayerSteps` | Existing pixi/npm/cargo/aur multi-stage + inline blocks |

## Cache-mount inheritance

Cache mounts come from `distro.yml` (format caches) and `builder.yml` (builder caches). `ov tasks.go` picks the right set based on task context:

| Task context | Cache mount(s) |
|---|---|
| `cmd:` as root | Distro format caches (`/var/cache/libdnf5` / `/var/cache/apt` / `/var/cache/pacman/pkg`) + `/ctx` bind mount |
| `cmd:` as non-root | `/tmp/npm-cache` (UID-scoped) + `/ctx` bind mount |
| `download:` | `/tmp/downloads` (shared across layers) |
| Package install | Distro format caches from `distro.yml` |
| pixi builder | `/tmp/pixi-cache` + `/tmp/rattler-cache` (UID-scoped) |
| npm builder | `/tmp/npm-cache` (UID-scoped) |

The `/ctx` bind mount exposes the layer's own directory tree to `cmd:` tasks — so you can still reference `/ctx/<file>` inside an escape-hatch shell block for one-off file access (though `copy:` / `write:` are strongly preferred).

## USER emission

- `user: root` → `USER 0`
- `user: ${USER}` → `USER <numeric UID>` (matches the pre-refactor convention; avoids `/etc/passwd` dependency at the switch point)
- `user: <uid>:<gid>` → `USER <uid>:<gid>` (e.g. `1010:1010`)
- `user: <name>` → `USER <name>` (literal; requires user to exist — create via earlier `cmd: useradd` task)

`COPY --chown=` uses numeric `<UID>:<GID>` for `${USER}` (BuildKit-safe), name-pairs for literal users.

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
- Multi-stage builds use builder images from `builder.yml` (`pixi-builder`, `npm-builder`, `archlinux-builder` for AUR, etc.).
- Stale `.build/<image>/` directories (from removed or renamed images) are cleaned at the start of each generation.

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
- `/ov-dev:generate` — Deep dive on Containerfile emission internals, `Task` struct, per-verb emitters, `stageInlineContent`.
- `/ov-dev:go` — Source-code map: `ov/tasks.go` (376 lines), `ov/generate.go:writeLayerSteps`, `ov/layers.go` struct definitions.
