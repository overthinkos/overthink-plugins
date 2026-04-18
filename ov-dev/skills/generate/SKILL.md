---
name: generate
description: |
  Containerfile generation: understanding ov image generate output, multi-stage builds,
  intermediate images, and the .build/ directory. Use when debugging or understanding
  generated Containerfiles.
  MUST be invoked before reading or modifying any Go source file in ov/.
---

# Generate - Containerfile Generation

## Overview

`ov image generate` reads `image.yml` and `layers/`, resolves dependency graphs, and writes Containerfiles to `.build/`. Generation is idempotent and `.build/` is disposable (gitignored). Understanding the generated output is essential for debugging build issues.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Generate all | `ov image generate` | Write .build/ (Containerfiles) |
| Generate with tag | `ov image generate --tag v1.0.0` | Override CalVer tag |
| List targets | `ov image list targets` | Build targets in dependency order |
| Inspect config | `ov image inspect <image>` | Show resolved config JSON |

## Generated Output Structure

```
.build/
├── <image>/
│   ├── Containerfile              # One per image
│   ├── supervisor/*.conf          # Supervisord configs (if service layers)
│   └── traefik-routes.yml         # Traefik config (if route layers)
└── _layers/<name>                 # Symlinks to remote layers
```

## Containerfile Structure

The generated Containerfile follows this order:

1. **Multi-stage build stages** — scratch stages per layer, builder stages from `build.yml` `builder:` templates (pixi, npm, aur, cargo), init system config assembly (driven by `build.yml` `init:` section), traefik routes
2. **`FROM ${BASE_IMAGE}`** — external bases get bootstrap from `build.yml` `distro:` section (install cmd, cache mounts, workarounds); internal bases get `USER root`
3. **Image metadata** — consolidated `ENV` directives, `EXPOSE` ports, `org.overthinkos.*` labels
4. **COPY build artifacts** — config-driven from `build.yml` `builder.<name>.copy_artifacts` and `copy_binary` definitions
5. **Per-layer install steps** — see "Task emission pipeline" below. `USER` toggles as each task's `user:` field requires.
6. **Final assembly** — init system config assembly, traefik routes COPY, `USER <UID>`, `RUN bootc container lint` (bootc images only)

**Config-driven generation:** All format-specific install commands, cache mounts, repo setup, builder stages, and init fragments are defined in `build.yml` at the project root as Go `text/template` strings — three top-level sections: `distro:`, `builder:`, `init:`. Each distro entry contains both bootstrap config and its package format definitions. Referenced via `format_config: build.yml` in `image.yml` — supports local paths and remote `@github.com/org/repo/path:version` refs. Adding a new format (e.g., `apk` for Alpine) requires only YAML changes — zero Go code modifications.

## Task emission pipeline

All install-task logic lives in a single file: `ov/tasks.go` (~380 lines). Authored layer-side as `tasks:` list in `layer.yml`; emitted as Containerfile directives via this sequence per layer inside `writeLayerSteps` (`ov/generate.go:1021`):

```
1. # Layer: <name>                 (comment header)
2. ARG TARGETARCH + ENV ARCH=${TARGETARCH}    (once; from emitVarsEnv)
3. ENV <K>=<V> for each layer.Vars entry      (also from emitVarsEnv)
4. Package install: rpm/deb/pac/aur  (unchanged — format template render)
5. emitTasks(b, layer, img, buildDir, contextRelPrefix, initialUser):
   for each t in layer.tasks:
     resolve ${VAR} in non-verbatim fields
     user := resolveUserSpec(t.User, img)   // numeric UID for ${USER}
     if user != runningUser: emit USER <value>
     switch t.Kind():
       case "mkdir":    emitMkdirBatch  (coalesces adjacent same-user+same-mode)
       case "copy":     emitCopy        (COPY --from=<layer> --chmod= --chown=)
       case "write":    stageInlineContent + emitWrite (COPY from .build _inline)
       case "link":     emitLinkBatch   (coalesces adjacent same-user)
       case "download": emitDownload    (RUN curl + extractor + /tmp/downloads cache)
       case "setcap":   emitSetcapBatch (coalesces; strip on empty caps)
       case "cmd":      emitCmd         (RUN bash -c 'set -e; ...' + /ctx bind)
       case "build":    writeLayerSteps handles builder placement
6. USER root reset (unless last layer + skipRootReset)
```

### `Task` struct (`ov/layers.go:195`)

Flat struct with verb-discriminator fields. Exactly one of `Cmd` / `Mkdir` / `Copy` / `Write` / `Link` / `Download` / `Setcap` / `Build` must be non-empty. Shared modifiers (`User`, `Mode`, `To`, `Target`, `Content`, `Extract`, `Include`, `Env`, `Caps`, `Comment`) are validated per-verb in `ov/validate.go:validateLayerTasks`.

```go
func (t *Task) Kind() (string, error)   // returns verb or error ("no action" / "conflicting actions")
```

### Emitter helpers (all in `ov/tasks.go`)

| Helper | Output |
|---|---|
| `emitVarsEnv(b, vars)` | `ARG TARGETARCH` + `ENV ARCH=${TARGETARCH}` + sorted `ENV K=V` |
| `emitMkdirBatch(b, []Task, img)` | One `RUN mkdir -p … [&& chmod <mode> …]` |
| `emitCopy(b, Task, layerStage, img)` | `COPY --from=<layerStage> --chmod= [--chown=] <src> <to>` |
| `emitWrite(b, Task, srcPath, img)` | `COPY [--chown=] --chmod= <srcPath> <path>` where `srcPath` is the staged inline-content file |
| `emitLinkBatch(b, []Task, img)` | One `RUN ln -sf t1 l1 && ln -sf t2 l2 …` |
| `emitDownload(b, Task, img)` | `RUN --mount=type=cache,dst=/tmp/downloads bash -c 'export BUILD_ARCH=$(uname -m); curl … \| <extractor>'` |
| `emitSetcapBatch(b, []Task, img)` | `RUN setcap -r … && setcap caps path …` |
| `emitCmd(b, Task, layerStage, img, userIsRoot)` | `RUN --mount=type=bind,from=<layerStage>,source=/,target=/ctx [--mount=type=cache,…] bash -c $'BUILD_ARCH=$(uname -m)\nset -e\n<command>'` (ANSI-C `$'...'` quoting — see below) |

### Shell-quoting helpers (`ov/tasks.go`)

Two helpers in `tasks.go` handle the two shell-quoting problems that
come up when embedding scripts and JSON into a Containerfile:

| Helper | Purpose | Used by |
|--------|---------|---------|
| `shellSingleQuote(s)` | Standard `'...'` quoting with `'\''` escape for embedded single quotes. | `emitDownload` (for `t.Env` values), `writeJSONLabel` (for LABEL JSON values containing `awk '{…}'` etc.) |
| `shellAnsiQuote(s)` | Bash ANSI-C `$'...'` quoting: real newlines, tabs, and backslashes survive as `\n` / `\t` / `\\`. Keeps a multi-line script on a single physical line. | `emitCmd` body |

**Why `shellAnsiQuote` exists**: a plain `bash -c '<body>'` with embedded
real newlines gets cut off by podman's Dockerfile parser at the first
newline inside the quoted string — the parser is line-oriented and
treats everything after the newline as a new instruction. `$'…\n…'`
serializes the whole script to one physical line; bash then reassembles
the newlines when it parses the `-c` argument. Works on Fedora/Arch
(bash-linked `/bin/sh`) and Alpine (ash supports ANSI-C).

**Why `export VAR=val;` beats `VAR=val cmd`** in `emitDownload`: the
`VAR=val cmd` prefix form sets `VAR` in `cmd`'s environment, but bash
expands `${VAR}` in `cmd`'s arguments *before* the environment is
assembled. Result: the URL `pixi-${BUILD_ARCH}-unknown-linux-musl.tar.gz`
gets rendered as `pixi--unknown-linux-musl.tar.gz` (empty arch) →
404. Terminating with `;` forces bash to run the export statement first,
putting the value in scope for the downstream URL expansion.

### Inline-content staging

`stageInlineContent(buildDir, contextRelPrefix, layerName, content)` writes `write:` task content to `<buildDir>/_inline/<layer>/<sha256>` on disk and returns the build-context-relative path (e.g. `.build/<image>/_inline/<layer>/<sha256>`). Content-addressed filename makes writes idempotent — identical content writes no-op; changed content produces a new hash which invalidates only that COPY's cache layer. **No shell heredoc ever appears in the Containerfile** — content travels as bytes.

### User resolution

`resolveUserSpec(userField, img)` → `(directive, chownPair)`:

- `root` / `0` / empty → `("0", "")` (root is COPY's default; skip `--chown`)
- `${USER}` → `(strconv.Itoa(img.UID), fmt.Sprintf("%d:%d", img.UID, img.GID))` — **numeric** directive, matching the pre-refactor `USER <UID>` convention and avoiding `/etc/passwd` dependencies at the switch point
- `<uid>:<gid>` → pass through
- bare numeric `<uid>` → `(u, u + ":" + u)`
- literal name → `(u, u + ":" + u)` (COPY `--chown=<name>:<name>` works when the user exists in the image)

### Variable substitution

Two-tier:

- **Generate-time:** `taskSubstPath` expands `~/` to `img.Home` and substitutes `${USER}` / `${UID}` / `${GID}` / `${HOME}` literally (values known at generate time). Applied to paths, URLs, modes, `to`, `target`, etc.
- **Build-time (Docker):** `${ARCH}` comes from BuildKit's `TARGETARCH` via `ARG TARGETARCH` + `ENV ARCH=${TARGETARCH}` emitted at layer top. Layer-local `vars:` become `ENV` directives too. Docker substitutes these in COPY paths, RUN commands, and ENV values.
- **Build-time (shell):** `${BUILD_ARCH}` is auto-injected as a local shell variable at the top of each `cmd:` / `download:` RUN (`BUILD_ARCH=$(uname -m)`). Unlike `${ARCH}`, it isn't available in non-shell fields.

`taskUnresolvedRefs(s, known)` returns `${NAME}` references that don't resolve against auto-exports ∪ layer `vars:` — used by `validateLayerTasks` to error on typos.

### Adjacent-coalescing (`taskCoalescesWith`)

Only these verbs coalesce: `mkdir`, `link`, `setcap`. Coalescing requires same verb + same `User` field (literal equality before resolution). The orchestrator loop consumes consecutive matching tasks into a batch, then flushes as a single directive. Non-adjacent same-verb tasks never merge — reordering across other verbs is forbidden.

### Parent-dir auto-insertion

For `copy:` and `write:` tasks, if the destination's parent directory isn't already registered in `declaredDirs` (which tracks every `mkdir:` task PLUS all ancestor paths, matching Unix `mkdir -p` semantics), the orchestrator prepends a single `RUN mkdir -p <parent>` (as the task's `user:`) immediately before the COPY. This is the only implicit directive in the pipeline; authors can disable it for a specific path by declaring the parent via explicit `mkdir:`.

## Multi-Stage Builds

### Pixi Build Stage

```dockerfile
FROM ghcr.io/overthinkos/fedora-builder:2026.48.1808 AS supervisord-pixi-build
WORKDIR /home/user
COPY layers/supervisord/pixi.toml pixi.toml
RUN pixi install
```

Uses the configured builder image. No `apt-get install` needed — builder has pixi, node, npm, gcc, cmake, git.

### npm Build Stage

```dockerfile
FROM <builder> AS openclaw-npm-build
COPY layers/openclaw/package.json package.json
RUN npm install -g --prefix /npm-global
```

### AUR Build Stage (Arch Linux)

For layers with `aur:` packages, the generator creates a multi-stage build using the `builders.aur` image:

```dockerfile
FROM ghcr.io/overthinkos/archlinux-builder:2026.84.942 AS my-tool-aur-build
USER root
RUN echo 'user ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/builder
USER 1000
WORKDIR /home/user
RUN --mount=type=cache,dst=/var/cache/pacman/pkg,sharing=locked \
    mkdir -p /tmp/aur-build && \
    cp /etc/makepkg.conf /tmp/makepkg.conf && \
    sed -i '/^OPTIONS/s/ debug/ !debug/' /tmp/makepkg.conf && \
    yay -S --noconfirm --needed --builddir /tmp/aur-build --makepkgconf /tmp/makepkg.conf \
      aur-package && \
    mkdir -p /tmp/aur-pkgs && \
    find /tmp/aur-build -name '*.pkg.tar.zst' -exec cp {} /tmp/aur-pkgs/ \;
```

In the main image, the built packages are installed:
```dockerfile
COPY --from=my-tool-aur-build /tmp/aur-pkgs/ /tmp/aur-pkgs/
RUN --mount=type=cache,dst=/var/cache/pacman/pkg,sharing=locked \
    pacman -U --noconfirm /tmp/aur-pkgs/*.pkg.tar.zst && \
    rm -rf /tmp/aur-pkgs
```

Key details: passwordless sudo is required because yay calls `pacman -U` as root. Debug packages are disabled via a patched copy of `makepkg.conf` (the build user can't modify `/etc/` directly).

**Status:** Working. Verified with `yay-bin` in `arch-test` image.

### Scratch Context Stage

```dockerfile
FROM scratch AS mylib-ctx
COPY layers/mylib/ /
```

## LABEL Placement — Cache Efficiency

**All OCI LABEL directives are emitted at the end of the final stage**,
after the last `USER` directive. This is an intentional cache-efficiency
choice driven by `ov/generate.go`'s `writeLabels` call being placed
after `writeLayerSteps` + the final `USER` emission.

Why it matters: the `org.overthinkos.tests` LABEL is the most-volatile
piece of image metadata — it changes every time a test is added, edited,
or removed. If LABELs appeared BEFORE the RUN/COPY install steps (as
before the relocation), buildkit's cache invalidated at the first
changed LABEL and every downstream instruction rebuilt from scratch.
For a 138-step stack like `immich-ml`, that meant re-running pnpm
install, the Immich server build, geodata downloads — minutes to hours
for a one-line test edit.

With LABELs at the end, a test/label edit only re-runs the LABEL
instructions themselves (pure manifest metadata, no filesystem work).
Measured: ~2 seconds of delta over a no-change rebuild baseline of
~24 seconds for `filebrowser`.

LABELs are safe to move because they have no functional dependency on
subsequent instructions — they attach to the final image manifest and
are read only via `podman inspect` (an unordered map). None of the
runtime consumers (`ov config`, `ov start`, `ov status`, `ov test`,
`ov shell`, `ov alias install`) care about directive order.

### LABEL JSON escaping (`writeJSONLabel`)

`writeJSONLabel` in `ov/generate.go` routes every LABEL value through
`shellSingleQuote` before emitting `LABEL key=<quoted>`. This is
required because test/task commands often contain literal `'` characters
(e.g. `awk '{print $1}'`, `sed 's/foo/bar/'`) which JSON preserves
verbatim. Without escaping, the embedded quote terminates the LABEL's
own surrounding quote and the rest of the JSON blob is parsed as
`key=value` pairs — which typically fails with "can't find = in ..." on
the JSON payload.

### CalVer cascade (known residual cost)

Each `ov image build` assigns a fresh CalVer timestamp to every
intermediate image it produces. Downstream stages reference those by
their new tag (e.g. `FROM ghcr.io/overthinkos/fedora:2026.108.1955`),
which is a literally different image reference than last run. The
downstream stage's first `FROM` step therefore never hits cache on a
fresh run — but the subsequent content-addressed RUN/COPY steps do
hit cache, because buildkit's per-instruction key is content-based
from the FROM image ID onward, not from the tag string.

The LABELs-at-end fix wins on the common case (edit a test, rebuild the
same stack) and doesn't help the "fresh base-image build" case. A
content-hash tag mode would eliminate the residual FROM-step cost; not
yet implemented.

## Intermediate Images

Auto-generated at branch points where multiple images share common layer prefixes:

```
fedora (external)
  -> fedora-supervisord (auto: pixi + python + supervisord)
     -> fedora-test (adds: traefik, testapi)
     -> openclaw (adds: nodejs, openclaw)
```

`ComputeIntermediates()` uses a prefix trie to detect divergence points. `GlobalLayerOrder()` prioritizes popular layers for cache efficiency.

## User Resolution

Configurable via `user`, `uid`, `gid` fields in `image.yml` (defaults: `"user"`, 1000, 1000).

For external base images, `ov` calls `registry.go:InspectImageUser()` which:
1. Pulls the base image via go-containerregistry
2. Extracts `/etc/passwd` from image layers (top layer first)
3. Finds user at configured UID
4. If found: overrides username, home directory, and GID (e.g., `ubuntu` with home `/home/ubuntu` for `ubuntu:24.04`)
5. If not found: uses configured defaults, bootstrap creates the user

For internal base images, user context is inherited from the parent image.

The bootstrap conditionally creates the user:
```dockerfile
RUN getent passwd <UID> >/dev/null 2>&1 || \
    (getent group <GID> >/dev/null 2>&1 || groupadd -g <GID> <user> && \
     useradd -m -u <UID> -g <GID> -s /bin/bash <user>)
```

Source: `ov/registry.go` (`InspectImageUser`).

## OCI Labels

Built images embed runtime metadata as labels (prefix: `org.overthinkos.`), making images self-describing for runtime commands (`ov shell`, `ov start`, `ov config`, `ov alias install`).

| Label | Type | Example |
|-------|------|---------|
| `org.overthinkos.version` | string | `"1"` (schema version) |
| `org.overthinkos.image` | string | `"openclaw"` |
| `org.overthinkos.registry` | string | `"ghcr.io/overthinkos"` (omitted if empty) |
| `org.overthinkos.bootc` | string | `"true"` (omitted if false) |
| `org.overthinkos.uid` / `.gid` | string | `"1000"` |
| `org.overthinkos.user` / `.home` | string | `"user"` / `"/home/user"` |
| `org.overthinkos.ports` | JSON | `["18789:18789"]` |
| `org.overthinkos.volumes` | JSON | `[{"name":"data","path":"/home/user/.openclaw"}]` |
| `org.overthinkos.aliases` | JSON | `[{"name":"openclaw","command":"openclaw"}]` |
| `org.overthinkos.security` | JSON | `{"privileged":false,"cap_add":["SYS_PTRACE"]}` |
| `org.overthinkos.network` | string | `"host"` (omitted if default) |
| ~~`org.overthinkos.tunnel`~~ | | Removed — tunnel is a deploy-time concern (deploy.yml only) |
| `org.overthinkos.fqdn` | string | FQDN for tunnel routing |
| `org.overthinkos.acme_email` | string | ACME certificate email |
| `org.overthinkos.env` | JSON | `["KEY=VALUE"]` runtime env vars |
| `org.overthinkos.hooks` | JSON | lifecycle hooks config |
| `org.overthinkos.vm` | JSON | VM config (bootc images) |
| `org.overthinkos.libvirt` | JSON | libvirt XML snippets |
| `org.overthinkos.routes` | JSON | `[{"host":"app.localhost","port":8080}]` |
| `org.overthinkos.init` | string | active init system name |
| `org.overthinkos.services.<init>` | JSON | service names per init system |
| `org.overthinkos.env_layers` | JSON | layer-level env vars (merged) |
| `org.overthinkos.path_append` | JSON | PATH append entries |
| `org.overthinkos.engine` | string | Required run engine (`docker`/`podman`, omitted if any) |
| `org.overthinkos.port_protos` | JSON | `{"9222":"tcp"}` port protocol overrides (non-http only) |
| `org.overthinkos.port_relay` | JSON | `[9222]` ports with socat relay |
| `org.overthinkos.status` | string | Effective status: `working`, `testing`, or `broken` (always emitted) |
| `org.overthinkos.info` | string | Aggregated status info from image + non-working layers (omitted if empty) |
| `org.overthinkos.layer_versions` | JSON | `{"chrome":"2026.83.1430"}` layer name → CalVer (only versioned layers) |
| `org.overthinkos.skills` | string | Skill documentation URL (omitted if no skill exists) |

Volumes use short names in labels (prefix `ov-<image>-` added at runtime). Empty arrays are omitted. JSON built from sorted slices for cache stability. Runtime commands read OCI labels exclusively (via `ExtractMetadata` in `ov/labels.go`) plus `deploy.yml` overlay — they never touch `image.yml` at runtime. That's why `ov shell myimage` works from any directory as long as the image is in local storage (if not, `ExtractMetadata` returns `ErrImageNotLocal` and the CLI suggests `ov image pull`). See `/ov:image` for the build/deploy boundary and `/ov:pull` for the sentinel pattern. Labels also include `org.overthinkos.init` for init system identification and `org.overthinkos.services.<init>` for per-init service lists.

Source: `ov/labels.go`, `ov/generate.go` (`writeLabels`).

## Runtime-Only Features

Security configuration (`security:` in layer.yml/image.yml) and environment variable injection (`env:`, `env_file:`) are **runtime-only** features. They affect container run arguments (`--privileged`, `--cap-add`, `-e`) but do not appear in generated Containerfiles.

## Cache Mounts

| Emission site | Cache path | Options |
|---|---|---|
| `rpm.packages` | `/var/cache/libdnf5` | `sharing=locked` |
| `deb.packages` | `/var/cache/apt` + `/var/lib/apt` | `sharing=locked` |
| `pac.packages` + `aur` | `/var/cache/pacman/pkg` | `sharing=locked` |
| `cmd:` as root | Distro-format caches (above) + `/ctx` bind to layer stage | — |
| `cmd:` as non-root | `/tmp/npm-cache` (UID-scoped) + `/ctx` bind to layer stage | `uid=<UID>,gid=<GID>` |
| `download:` | `/tmp/downloads` (shared across layers) | — |
| pixi builder stage | `/tmp/pixi-cache` + `/tmp/rattler-cache` | `uid=<UID>,gid=<GID>` |
| npm builder stage | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |
| cargo inline | `/tmp/cargo-cache` | `uid=<UID>,gid=<GID>` |

UID/GID in cache mounts are dynamic (from resolved image config). All non-root cache mounts use flat `/tmp/<tool>-cache` paths to avoid buildah permission issues with nested paths.

## Common Workflows

### Debug Why a Build Fails

```bash
ov image generate                              # Generate Containerfiles
cat .build/my-image/Containerfile        # Inspect the generated Containerfile
ov image validate                              # Check for validation errors
ov image inspect my-image                      # See resolved config
```

### Understand Layer Ordering

```bash
ov image list targets                          # Shows dependency-ordered build sequence
ov image inspect my-image --format layers      # Shows layer list for an image
```

## Cross-References

- `/ov:layer` — **Canonical author-facing reference** for the task verb catalog, `vars:` substitution, YAML anchors, execution order. The emitter pipeline here implements what's documented there.
- `/ov:generate` — User-facing `ov image generate` command.
- `/ov-dev:go` — Source code map: `ov/tasks.go` (emitter pipeline), `ov/generate.go:writeLayerSteps` (orchestrator call site), `ov/layers.go:Task` struct, `ov/validate.go:validateLayerTasks`.
- `/ov:validate` — User-facing validation rules (what `validateLayerTasks` enforces).
- `/ov:build` — Building from generated Containerfiles.
- Source: `ov/generate.go`, `ov/tasks.go`, `ov/intermediates.go`, `ov/graph.go`.

## When to Use This Skill

**MUST be invoked** before reading or modifying Go source files. Invoke this skill BEFORE launching Explore agents on ov/ code.
