---
name: generate-source
description: |
  Containerfile generation: understanding ov box generate output, multi-stage builds,
  intermediate images, and the .build/ directory. Use when debugging or understanding
  generated Containerfiles.
  MUST be invoked before reading or modifying any Go source file in ov/.
---

# Generate - Containerfile Generation

## Overview

`ov box generate` reads `box.yml` and `candy/`, resolves dependency graphs, and writes Containerfiles to `.build/`. Generation is idempotent and `.build/` is disposable (gitignored). Understanding the generated output is essential for debugging build issues.

Generation runs through the shared `DeployTarget` interface. `OCITarget` is the build-mode implementation; it consumes the `InstallPlan` IR emitted by `BuildDeployPlan` and writes Containerfile text. `PodDeployTarget` and `LocalDeployTarget` are the deploy-mode siblings consuming the same IR. For the IR shape and step kinds, see **`/ov-internals:install-plan`**. For local-deploy supporting files (ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref), see **`/ov-internals:local-infra`**.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Generate all | `ov box generate` | Write .build/ (Containerfiles) |
| Generate with tag | `ov box generate --tag v1.0.0` | Override CalVer tag |
| List targets | `ov box list targets` | Build targets in dependency order |
| Inspect config | `ov box inspect <image>` | Show resolved config JSON |

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

**Config-driven generation:** All format-specific install commands, cache mounts, repo setup, builder stages, and init fragments are defined in `build.yml` at the project root as Go `text/template` strings — three top-level sections: `distro:`, `builder:`, `init:`. Each distro entry contains both bootstrap config and its package format definitions. Referenced via `format_config: build.yml` in `box.yml` — supports local paths and remote `@github.com/org/repo/path:version` refs. Adding a new format (e.g., `apk` for Alpine) requires only YAML changes — zero Go code modifications.

## Task emission pipeline

All install-task logic lives in a single file: `ov/tasks.go` (~380 lines). Authored layer-side as `task:` list in `candy.yml`; emitted as Containerfile directives via this sequence per layer inside `writeLayerSteps` (`ov/generate.go:1021`):

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

The `Layer` struct that feeds emission carries its `require:` / `layer:` refs as `[]CandyRef` (one typed list each — no parallel bare/raw arrays), and its `Has*` predicates (`HasEnv`/`HasPorts`/`HasVolumes`/…) are derived methods (only the filesystem-probe caches like `HasPixiToml` stay fields). Layers are populated by the single `populateLayerFromYAML`. Which layers reach this emission pipeline is decided upstream by the reachability-scoped, per-entity-version resolver (the git tag is the fetch coordinate; the layer's own `version:` is the identity) — see `/ov-internals:go` "Remote-layer resolver".

### Emitter helpers (all in `ov/tasks.go`)

| Helper | Output |
|---|---|
| `emitVarsEnv(b, vars)` | `ARG TARGETARCH` + `ENV ARCH=${TARGETARCH}` + sorted `ENV K=V` |
| `emitMkdirBatch(b, []Task, img)` | One `RUN mkdir -p … [&& chmod <mode> …]` |
| `emitCopy(b, Task, layerStage, img)` | `COPY --from=<layerStage> --chmod= [--chown=] <src> <to>` |
| `emitWrite(b, Task, srcPath, img)` | `COPY [--chown=] --chmod= <srcPath> <path>` where `srcPath` is the staged inline-content file |
| `emitLinkBatch(b, []Task, img)` | One `RUN ln -sf t1 l1 && ln -sf t2 l2 …` |
| `emitDownload(b, Task, img)` | `RUN --mount=type=cache,dst=/tmp/downloads [+ task cache:] bash -c '… curl -o "$__c.part" && mv → /tmp/downloads/<sha256>; <extractor> "$__c"'` — content-addressed, fetch-once, integrity-safe |
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
- `${USER}` → `(strconv.Itoa(img.UID), fmt.Sprintf("%d:%d", img.UID, img.GID))` — **numeric** `USER <UID>` directive, avoiding `/etc/passwd` dependencies at the switch point
- `<uid>:<gid>` → pass through
- bare numeric `<uid>` → `(u, u + ":" + u)`
- literal name → `(u, u + ":" + u)` (COPY `--chown=<name>:<name>` works when the user exists in the image)

### Variable substitution

Two-tier:

- **Generate-time:** `taskSubstPath` expands `~/` to `img.Home` and substitutes `${USER}` / `${UID}` / `${GID}` / `${HOME}` literally (values known at generate time). Applied to paths, URLs, modes, `to`, `target`, etc.
- **Build-time (Docker):** `${ARCH}` comes from BuildKit's `TARGETARCH` via `ARG TARGETARCH` + `ENV ARCH=${TARGETARCH}` emitted at layer top. Layer-local `var:` become `ENV` directives too. Docker substitutes these in COPY paths, RUN commands, and ENV values.
- **Build-time (shell):** `${BUILD_ARCH}` is auto-injected as a local shell variable at the top of each `cmd:` / `download:` RUN (`BUILD_ARCH=$(uname -m)`). Unlike `${ARCH}`, it isn't available in non-shell fields.

`taskUnresolvedRefs(s, known)` returns `${NAME}` references that don't resolve against auto-exports ∪ layer `var:` — used by `validateLayerTasks` to error on typos.

### Adjacent-coalescing (`taskCoalescesWith`)

Only these verbs coalesce: `mkdir`, `link`, `setcap`. Coalescing requires same verb + same `User` field (literal equality before resolution). The orchestrator loop consumes consecutive matching tasks into a batch, then flushes as a single directive. Non-adjacent same-verb tasks never merge — reordering across other verbs is forbidden.

### Parent-dir auto-insertion

For `copy:` and `write:` tasks, if the destination's parent directory isn't already registered in `declaredDirs` (which tracks every `mkdir:` task PLUS all ancestor paths, matching Unix `mkdir -p` semantics), the orchestrator prepends a single `RUN mkdir -p <parent>` (as the task's `user:`) immediately before the COPY. This is the only implicit directive in the pipeline; authors can disable it for a specific path by declaring the parent via explicit `mkdir:`.

## Multi-Stage Builds

### Pixi Build Stage

```dockerfile
FROM ghcr.io/overthinkos/fedora-builder:2026.48.1808 AS supervisord-pixi-build
WORKDIR /home/user
COPY candy/supervisord/pixi.toml pixi.toml
RUN pixi install
```

Uses the configured builder image. No `apt-get install` needed — builder has pixi, node, npm, gcc, cmake, git.

### npm Build Stage

```dockerfile
FROM <builder> AS openclaw-npm-build
COPY candy/openclaw/package.json package.json
RUN npm install -g --prefix /npm-global
```

### AUR Build Stage (Arch Linux)

For layers with `aur:` packages, the generator creates a multi-stage build using the `builders.aur` image:

```dockerfile
FROM ghcr.io/overthinkos/arch-builder:2026.84.942 AS my-tool-aur-build
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
COPY candy/mylib/ /
```

## Auto-Intermediate Images — grouping by `(Base, UID)`

Sibling images that share the same base are grouped by
`ComputeIntermediates` in `ov/intermediates.go` so repeated layer
prefixes become shared intermediate images (the `fedora-nonfree-ssh-client-dbus`
pattern). The grouping key is `siblingKey{base, uid}`, not a bare base
string: grouping on base alone would collapse images with **different
UIDs** into one intermediate whose resolved `HOME` is always the
default (`/home/user`), so a child image with `uid: 0` (`/home/root`)
would inherit `/home/user/.pixi/bin` as dead `ENV PATH` entries it
can't override.

Per-UID sibling groups get their own intermediate chains. When the UID
differs from `cfg.Defaults.UID` (typically 1000), `pickAutoName`
appends `-uid<N>` to the intermediate name so OCI tags don't collide
with the default-UID variant.

```go
// ov/intermediates.go (abridged)
type siblingKey struct {
    base string
    uid  int
}
siblingGroups := make(map[siblingKey][]string)
for name, img := range images {
    if builderNames[name] {
        continue
    }
    k := siblingKey{img.Base, img.UID}
    siblingGroups[k] = append(siblingGroups[k], name)
}
```

`createIntermediate()` then derives `User/GID/Home` from the group's
UID (uid=0 → `root`/`/root`; else default user + `/home/<user>`)
instead of always using defaults. This keeps HOME-relative
`env:` / `path_append:` expansion consistent with the child images.

Blast radius: uid=1000 images (the overwhelming majority) share their
intermediates as usual. Uid=0 images get their own `-uid0`-suffixed
intermediates. The ecosystem's coder/ov images run as uid=1000 + sudo
(see `/ov-coder:fedora-coder`), so the per-UID split is dormant
protection against any future uid=0 image — it stays in place because
the defensive grouping is cheap (one extra struct field) and prevents
a whole class of silent PATH regressions.

Covered by `ov/intermediates_test.go` (the test images construct
`ResolvedImage{}` without an explicit UID, so they trip `uid=0` →
`-uid0` naming).

## LABEL Placement — Cache Efficiency

**All OCI LABEL directives are emitted at the end of the final stage**,
after the last `USER` directive. This is an intentional cache-efficiency
choice driven by `ov/generate.go`'s `writeLabels` call being placed
after `writeLayerSteps` + the final `USER` emission.

Why it matters: the `org.overthinkos.eval` LABEL is the most-volatile
piece of image metadata — it changes every time a test is added, edited,
or removed. If LABELs appeared BEFORE the RUN/COPY install steps,
buildkit's cache would invalidate at the first changed LABEL and every
downstream instruction would rebuild from scratch. For a 138-step stack
like `immich-ml`, that means re-running pnpm install, the Immich server
build, geodata downloads — minutes to hours for a one-line test edit.

With LABELs at the end, a test/label edit only re-runs the LABEL
instructions themselves (pure manifest metadata, no filesystem work).
Measured: ~2 seconds of delta over a no-change rebuild baseline of
~24 seconds for `filebrowser`.

LABELs are safe to move because they have no functional dependency on
subsequent instructions — they attach to the final image manifest and
are read only via `podman inspect` (an unordered map). None of the
runtime consumers (`ov config`, `ov start`, `ov status`, `ov eval live`,
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

### CalVer tag shifts do NOT invalidate cache

Each `ov box build` generate assigns a fresh CalVer timestamp and
emits it as the default of `ARG BASE_IMAGE=<registry>/<name>:<calver>`
at the top of every intermediate Containerfile. The ARG default
appears in the Containerfile text but is NOT part of the cache key
for subsequent RUN/COPY steps.

Concretely: podman/buildah resolves the `FROM ${BASE_IMAGE}` step
first — turning the tag into an image SHA — then keys every later
instruction off that SHA plus the instruction text plus COPY source
content. If the SHA is the same as a cached build's parent (because
the upstream image's content is unchanged), the whole rest of the
build cache-hits. A CalVer bump on an unchanged upstream is free.

The `org.overthinkos.version` LABEL baked into each image is likewise NOT a
per-build timestamp — it's the content-derived `EffectiveVersion` (the image's
dedicated `version:`, else the highest layer `version:` across the chain; computed
by `computeEffectiveVersions` in `ov/effective_version.go`). It is STABLE across
builds when no layer changed, so it does not shift the image's config → SHA and
therefore does not cascade cache-misses to children via `FROM`. Only a real
content change (a bumped layer `version:`) moves it.

Cache-miss only happens when something in the build input genuinely
changes: the parent image's content (different SHA resolved by the
FROM), a layer's scratch-stage content (`COPY candy/<name>/ /`
source bytes), or an instruction's text (package list, cmd body).
See `/ov-build:build` "Cache Efficiency" for the full list.

## Intermediate Images

Auto-generated at branch points where multiple images share common layer prefixes:

```
fedora (external)
  -> fedora-supervisord (auto: pixi + python + supervisord)
     -> fedora-test (adds: traefik, testapi)
     -> openclaw (adds: nodejs, openclaw)
```

`ComputeIntermediates()` uses a prefix trie to detect divergence points. `GlobalLayerOrder()` prioritizes popular layers for cache efficiency.

### Parent-vs-defaults inheritance (critical)

`createIntermediate()` must inherit `Distro` and `BuildFormats` **from the parent image first**, with `cfg.Defaults.*` as the fallback only when the parent is external or empty. The inverse — defaults winning over an explicit parent — is a silent-failure bug: an `arch`-rooted auto-intermediate would get re-tagged as `build: [rpm]` (because `defaults.Build=[rpm]` in `box.yml`), then the layer emitter would scan each layer for an `rpm:` section and find nothing (the layers only declare `pac:`) — emitting empty RUN steps. Symptom: the intermediate built fine but shipped without any of its layers' packages (e.g. `arch-ssh-client` without direnv/gnupg/openssh).

The current code resolves `inheritedDistro` / `inheritedBuilds` from the parent first and only falls back to defaults when both are empty. Regression guard: `TestComputeIntermediates_InheritDistroFromParent` constructs a config with `defaults.Build=[rpm]` but an `arch` parent carrying `[pac]`, and asserts that the auto-intermediate comes out `[pac]`. See `/ov-internals:go` `intermediates.go` row for the file-level note.

## User Resolution

Configurable via `user`, `uid`, `gid`, and `user_policy` fields in `box.yml` (defaults: `"user"`, 1000, 1000, `"auto"`).

### Declarative adopt: `build.yml distro.<name>.base_user:`

Some base images ship a pre-existing uid-1000 account (Ubuntu 24.04 ships `ubuntu:ubuntu`). The `base_user:` declaration in `build.yml` tells ov about it:

```yaml
distro:
  ubuntu:
    base_user: { name: ubuntu, uid: 1000, gid: 1000, home: /home/ubuntu }
```

`ov/config.go:ResolveImage` reconciles this against the image's `user_policy:`:

- `auto` (default) — adopt `base_user` when declared AND image didn't explicitly set `user:`.
- `adopt` — always adopt; hard-errors without a declaration.
- `create` — always create via useradd.

When adopt fires, `resolved.User` / `UID` / `GID` / `Home` are overwritten and `resolved.UserAdopted = true`.

### writeBootstrap — two emission branches

`ov/generate.go:writeBootstrap` keys on `img.UserAdopted`:

**Adopt (no useradd):**
```dockerfile
# User ubuntu (uid=1000) adopted from base image (declared in build.yml distro.base_user) — no useradd needed

WORKDIR /home/ubuntu
USER 1000
```

**Create (idempotent useradd):**
```dockerfile
RUN if ! getent passwd <UID> >/dev/null 2>&1; then \
      (getent group <GID> >/dev/null 2>&1 || groupadd -g <GID> <user>) && \
      useradd -m -u <UID> -g <GID> -s /bin/bash <user>; \
    fi

WORKDIR /home/<user>
USER <UID>
```

The create branch uses the `if !` form (rather than a `getent passwd <UID> >/dev/null 2>&1 ||` short-circuit) because it composes better with the adopt branch — no risk of silently short-circuiting on an unexpectedly-pre-existing user.

### Fallback path: `registry.go:InspectImageUser`

Pulls the base image via go-containerregistry and extracts `/etc/passwd` to auto-detect the home dir. The declarative `base_user:` approach is preferred (simpler, no network round-trip, explicit intent); `InspectImageUser` remains as a fallback for bases without a `base_user:` declaration.

Source: `ov/config.go:ResolveImage` (policy reconciliation), `ov/generate.go:writeBootstrap` (emission), `ov/format_config.go:BaseUserDef` (struct), `ov/registry.go:InspectImageUser` (fallback).

## Tag-section install emission

Distro-version tag sections (`debian:13:`, `ubuntu:24.04:`, etc.) are resolved via first-match-wins on the image's `distro:` priority list. Tag sections use the primary format's **full** install template — so they can carry `repos:`, `keys:`, `options:`, and `package:`, not just packages alone. The generator reads each tag section's full YAML map via `ov/layers.go:TagPkgConfig.Raw`.

## OCI Labels

Built images embed runtime metadata as labels (prefix: `org.overthinkos.`), making images self-describing for runtime commands (`ov shell`, `ov start`, `ov config`, `ov alias install`).

| Label | Type | Example |
|-------|------|---------|
| `org.overthinkos.version` | string | content-derived `EffectiveVersion` (the image's dedicated `version:`, else the highest layer `version:` across the chain — NOT the per-build tag), e.g. `"2026.144.1443"`. Resolution prefers this label over the tag (`local_image.go`); also the "is this an ov image?" presence sentinel read by `ExtractMetadata` |
| `org.overthinkos.image` | string | `"openclaw"` |
| `org.overthinkos.registry` | string | `"ghcr.io/overthinkos"` (omitted if empty) |
| `org.overthinkos.bootc` | string | `"true"` (omitted if false) |
| `org.overthinkos.uid` / `.gid` | string | `"1000"` |
| `org.overthinkos.user` / `.home` | string | `"user"` / `"/home/user"` |
| `org.overthinkos.port` | JSON | `["18789:18789"]` |
| `org.overthinkos.volume` | JSON | `[{"name":"data","path":"/home/user/.openclaw"}]` |
| `org.overthinkos.alias` | JSON | `[{"name":"openclaw","command":"openclaw"}]` |
| `org.overthinkos.security` | JSON | `{"privileged":false,"cap_add":["SYS_PTRACE"]}` |
| `org.overthinkos.network` | string | `"host"` (omitted if default) |
| `org.overthinkos.fqdn` | string | FQDN for tunnel routing |
| `org.overthinkos.acme_email` | string | ACME certificate email |
| `org.overthinkos.env` | JSON | `["KEY=VALUE"]` runtime env vars |
| `org.overthinkos.hook` | JSON | lifecycle hooks config |
| `org.overthinkos.vm` | JSON | VM config (bootc images) |
| `org.overthinkos.libvirt` | JSON | libvirt XML snippets |
| `org.overthinkos.route` | JSON | `[{"host":"app.localhost","port":8080}]` |
| `org.overthinkos.init` | string | active init system name |
| `org.overthinkos.service.<init>` | JSON | service names per init system |
| `org.overthinkos.env_layer` | JSON | layer-level env vars (merged) |
| `org.overthinkos.path_append` | JSON | PATH append entries |
| `org.overthinkos.engine` | string | Required run engine (`docker`/`podman`, omitted if any) |
| `org.overthinkos.platform.distro` | JSON | `["arch"]` distro identity (first match picks bootstrap/format templates) |
| `org.overthinkos.platform.format` | JSON | `["pac"]` package formats installed (`pac`, `rpm`, `deb`, `pixi`, `aur`, …) |
| `org.overthinkos.builder.use` | JSON | `{"aur":"arch-builder","pixi":"default-builder"}` consumer-side routing: format → builder image |
| `org.overthinkos.builder.provide` | JSON | `["pac","aur"]` producer-side capability: formats this image can build for others (builder images only) |
| `org.overthinkos.port_proto` | JSON | `{"9222":"tcp"}` port protocol overrides (non-http only) |
| `org.overthinkos.port_relay` | JSON | `[9222]` ports with socat relay |
| `org.overthinkos.status` | string | Effective status: `working`, `testing`, or `broken` (always emitted) |
| `org.overthinkos.info` | string | Aggregated status info from image + non-working layers (omitted if empty) |
| `org.overthinkos.layer_version` | JSON | `{"chrome":"2026.83.1430"}` layer name → CalVer (only versioned layers) |
| `org.overthinkos.skill` | string | Skill documentation URL (omitted if no skill exists) |

Tunnel configuration is NOT an OCI label — it is a deploy-time concern carried in `deploy.yml` only.

Volumes use short names in labels (prefix `ov-<image>-` added at runtime). Empty arrays are omitted. JSON built from sorted slices for cache stability. Runtime commands read OCI labels exclusively (via `ExtractMetadata` in `ov/labels.go`) plus `deploy.yml` overlay — they never touch `box.yml` at runtime. That's why `ov shell myimage` works from any directory as long as the image is in local storage (if not, `ExtractMetadata` returns `ErrImageNotLocal` and the CLI suggests `ov box pull`). See `/ov-image:image` for the build/deploy boundary and `/ov-build:pull` for the sentinel pattern. Labels also include `org.overthinkos.init` for init system identification and `org.overthinkos.service.<init>` for per-init service lists.

Source: `ov/labels.go`, `ov/generate.go` (`writeLabels`).

## Runtime-Only Features

Security configuration (`security:` in candy.yml/box.yml) and environment variable injection (`env:`, `env_file:`) are **runtime-only** features. They affect container run arguments (`--privileged`, `--cap-add`, `-e`) but do not appear in generated Containerfiles.

## Cache Mounts

| Emission site | Cache path | Options |
|---|---|---|
| `rpm.packages` | `/var/cache/libdnf5` | `sharing=locked` |
| `deb.packages` | `/var/cache/apt` + `/var/lib/apt` | `sharing=locked` |
| `pac.packages` + `aur` | `/var/cache/pacman/pkg` | `sharing=locked` |
| `cmd:` as root | Distro-format caches (above) + `/ctx` bind to layer stage + any `cache:` (shared) | — |
| `cmd:` as non-root | `/tmp/npm-cache` (UID-scoped) + `/ctx` bind to layer stage + any `cache:` (owned) | `uid=<UID>,gid=<GID>` |
| `download:` | `/tmp/downloads` (shared) — **content-addressed**: file fetched once, reused across builds + any `cache:` | — |
| `task: cache:` | each declared path; ownership from the task `user:` (root → shared/locked, else uid/gid-owned) | per-user |
| AUR builder stage | `/var/cache/pacman/pkg` (shared) + `/tmp/aur-srcdest` + `/tmp/aur-xdg-cache` (owned: makepkg SRCDEST + yay clones) | mixed |
| pixi builder stage | `/tmp/pixi-cache` + `/tmp/rattler-cache` | `uid=<UID>,gid=<GID>` |
| npm builder stage | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |
| cargo inline | `/tmp/cargo-cache` | `uid=<UID>,gid=<GID>` |

UID/GID in cache mounts are dynamic (from resolved image config). All non-root cache mounts use flat `/tmp/<tool>-cache` paths to avoid buildah permission issues with nested paths.

### Download caching + the generic `cache:` modifier (`emitDownload` / `taskCacheMounts`)

`emitDownload` (`ov/tasks.go`) writes every `download:` to a **content-addressed**
file in the `/tmp/downloads` cache mount: `__c=/tmp/downloads/$(printf %s "$url"
| sha256sum | cut -c1-64)`, fetched only when absent (`[ -s "$__c" ] ||`), via
`curl -o "$__c.part" && mv -f "$__c.part" "$__c"` (atomic rename — a
partial/corrupt download is never reused), then the extractor reads `"$__c"`.
So a file fetched once survives an upstream layer cache-miss instead of
re-downloading (the prior code declared the mount but streamed curl past it).

`taskCacheMounts(t, img)` (`ov/tasks.go`) renders a task's layer-declared
`cache:` paths as cache-mount flags, ownership derived from the task's `user:`
via `resolveUserSpec` (root → `SharedCacheMount`, else → `OwnedCacheMount`).
Honored by `emitCmd` and `emitDownload`. Lets any task persist heavy
downloads/build artifacts the same way package caches do (config-driven; the
cache-USE logic lives in the candy.yml task body).

`CacheMountDef.Owned` + `RenderCacheMountsAuto` (`cacheMountsAuto` template
func) let a single builder `cache_mount` list mix shared (root, e.g. pacman) and
owned (user, e.g. makepkg SRCDEST / yay clones) entries — used by the `aur`
builder in `build.yml` to cache AUR source downloads + clones across builds.

## Common Workflows

### Debug Why a Build Fails

```bash
ov box generate                              # Generate Containerfiles
cat .build/my-image/Containerfile        # Inspect the generated Containerfile
ov box validate                              # Check for validation errors
ov box inspect my-image                      # See resolved config
```

### Understand Layer Ordering

```bash
ov box list targets                          # Shows dependency-ordered build sequence
ov box inspect my-image --format layers      # Shows layer list for an image
```

## Cross-References

- `/ov-image:layer` — **Canonical author-facing reference** for the task verb catalog, `var:` substitution, YAML anchors, execution order. The emitter pipeline here implements what's documented there.
- `/ov-build:generate` — User-facing `ov box generate` command.
- `/ov-internals:go` — Source code map: `ov/tasks.go` (emitter pipeline), `ov/generate.go:writeLayerSteps` (orchestrator call site), `ov/layers.go:Task` struct, `ov/validate.go:validateLayerTasks`.
- `/ov-build:validate` — User-facing validation rules (what `validateLayerTasks` enforces).
- `/ov-build:build` — Building from generated Containerfiles.
- Source: `ov/generate.go`, `ov/tasks.go`, `ov/intermediates.go`, `ov/graph.go`.

## When to Use This Skill

**MUST be invoked** before reading or modifying Go source files. Invoke this skill BEFORE launching Explore agents on ov/ code.
