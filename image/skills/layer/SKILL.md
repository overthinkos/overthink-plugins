---
name: layer
description: |
  MUST be invoked before any work involving: candy authoring, charly.yml, tasks, pixi.toml, package.json, Cargo.toml, or any file under candy/. This skill is the authoritative reference for the `task:` verb catalog, `var:` substitution, execution order, and per-verb validation. Every other skill defers here for install-schema questions.
---

# Layer - Layer Authoring

## Overview

A **candy** is a directory under `candy/<name>/` that installs a single concern. Candies are the building blocks of container images in opencharly. Each candy declares its packages, environment variables, services, volumes, and **install tasks** in a single `charly.yml` file.

There is **one YAML file per candy** for install logic — no separate Taskfiles. Everything an author needs to install flows through `task:` and auto-detected package manifests (`pixi.toml`, `package.json`, `Cargo.toml`).

## `directory:` — where the layer's config files live

A charly.yml's relative file references (`task.copy`, `task.write` inline paths, `data.src`, install-file probes like `pixi.toml` / `package.json`, service-file globs) resolve against **`directory:`**, which defaults to `.` (the directory containing charly.yml).

Use `directory:` to keep charly.yml and its supporting config files in separate directories:

```yaml
# candy/my-app/charly.yml
directory: ../../configs/my-app       # resolves relative to charly.yml's dir
rpm: { packages: [foo] }
task:
  - copy: policies.json                # found at configs/my-app/policies.json
```

Resolution rule:
- `""` or `"."` → same dir as charly.yml (the default)
- relative path → joined onto charly.yml's dir
- absolute path → used as-is

`charly box validate` fails when `directory:` points at a path that doesn't exist.

## Kind-keyed standalone form (`candy: {name, …}`)

Every charly.yml is self-describing: a single `candy:` wrapper with an explicit `name:` field + the body. This makes candy files bundle-mergeable — concatenate with `---` separators to form a single file containing many candies.

```yaml
# candy/chrome/charly.yml
candy:
  name: chrome
  directory: .
  rpm: { packages: [chromium] }
  task:
    - copy: policies.json
```

The runtime parser accepts only this kind-keyed form. `charly migrate` converts any legacy candy files to the canonical shape in a single idempotent pass.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Scaffold new candy | `charly box new candy <name>` | Create candy directory with starter `charly.yml` (see `/charly-build:new`) |
| Edit a candy field | `charly candy set <name> <dotpath> <value>` | Comment-preserving YAML edit by dot-path |
| Append rpm/deb/pac/aur packages | `charly candy add-rpm <name> <pkg…>` (plus `add-deb`, `add-pac`, `add-aur`) | Idempotent append; auto-upgrades scaffold's null `package:` to a sequence |
| Append a Gherkin acceptance scenario (Agent Driven Evaluation) | `charly candy add-scenario <name> <scenario> --given/--when/--then [--pod --tag]` | Idempotent append (dedupe by name) to the candy's `description.scenario`. Writes prose-only steps; bind a deterministic check verb by editing the step, or leave it prose for the agent grader. See `/charly-eval:eval` "Agent Driven Evaluation" |
| Write a free-form file (`pixi.toml`, `root.yml`, …) | `charly box write <rel-path> --content X` | Escape hatch for files the schema setters don't cover; guarded against `..` traversal |
| List all candies | `charly box list candies` | Show available candies from filesystem |
| List services | `charly box list services` | Candies with `service` in charly.yml |
| List volumes | `charly box list volumes` | Candies with `volumes` in charly.yml |
| List aliases | `charly box list aliases` | Candies with `aliases` in charly.yml |
| Validate | `charly box validate` | Check all candies and boxes |

Every editor verb above auto-becomes an MCP tool via Kong reflection (`candy.set`, `candy.add-rpm`, `box.write`, …) so an agent driving `charly mcp serve` can author candies from scratch over RPC without touching the filesystem directly. See `/charly-build:charly-mcp-cmd` "Authoring tools" for the worked end-to-end example, and `/charly-build:new` for the project / box / candy scaffolders that bootstrap the flow.

### Editing charly.yml via the CLI (no hand-edit required)

The `charly candy …` group edits `candy/<name>/charly.yml` through the `yaml.v3` Node API, so **comments and key order are preserved** across edits. Unlike unmarshal-then-marshal, nothing gets scrambled when an agent (or a shell script) touches the file:

```bash
# Append packages (idempotent; handles scaffold's null `package:` value):
charly candy add-rpm sshd openssh-server openssh-clients
charly candy add-deb sshd openssh-server
charly candy add-pac sshd openssh

# Set any field by dot-path (value is parsed as YAML):
charly candy set sshd env.SSHD_PORT 22
charly candy set sshd service.name sshd
charly candy set sshd port '["22:22"]'
charly candy set sshd require '[supervisord]'

# Free-form files (layer scripts, pixi.toml, root.yml, *.service):
charly box write candy/sshd/root.yml --content 'tasks:\n  - cmd: echo configured\n'
charly box cat candy/sshd/root.yml
```

Implementation: `charly/scaffold_cmds.go` (verbs) + `charly/yaml_setter.go` (`SetByDotPath`). Tested in `charly/yaml_setter_test.go` — the comment-preservation guarantee is explicitly exercised (leading file comments, sibling keys, and per-key inline comments all survive round trips). See `/charly-internals:go` "Implementation insights" for the full rationale.

## Install Surface (what a layer directory holds)

A candy directory can contain any combination of these:

| Artifact | Runs as | Purpose |
|---|---|---|
| `charly.yml` `distro:` map (+ top-level `package:` base) | root | System packages declared declaratively (see Package Surface below) |
| `charly.yml` `task:` list | per-task `user:` | Ordered install operations — the primary extension point (see catalog below) |
| `pixi.toml` / `pyproject.toml` / `environment.yml` | user (builder stage) | Python/conda packages. Multi-stage build. Only one per candy |
| `package.json` | user (builder stage) | npm packages — installed globally via `npm install -g` |
| `Cargo.toml` + `src/` | user (builder stage) | Rust crate — built via `cargo install --path` |
| `build.sh` | user (builder stage) | Optional post-install script for pixi candies. Runs in the pixi builder after `pixi install`. For build-time logic that can't be expressed in pixi.toml (C extension compilation, npm builds, binary patching). |

**Auto-detection:** The build system scans each candy directory for these files. `pixi.toml`, `pyproject.toml`, `environment.yml`, `package.json`, and `Cargo.toml` trigger automatic multi-stage builds — no manual install commands needed. Use `task:` only for things those manifests can't express (binary downloads, file copies from `/ctx`, inline config writes, post-install configuration, `go install`, etc.).

**Root vs user rule:** Pixi/npm/cargo builders always run as user — never as root. For `task:`, the `user:` field per task is explicit: `user: root` for system-wide changes, `user: ${USER}` for anything under `${HOME}`, or a literal username for custom users (must be created earlier in the same candy via a `cmd:` task).

---

## Task Verb Catalog

Every task in `task:` is a YAML map with **exactly one verb key** (the discriminator) plus optional sibling modifiers. The verb's value is the primary argument. `charly box validate` rejects tasks with zero verbs or multiple verbs.

| Verb | Value | Required modifiers | Optional modifiers | Purpose |
|---|---|---|---|---|
| `cmd:` | shell command (multi-line OK) | — | `user`, `comment` | Arbitrary shell — last-resort escape hatch |
| `mkdir:` | directory path | — | `user`, `mode`, `comment` | Create directory (`mkdir -p`; coalesces with adjacent) |
| `copy:` | source relative to layer dir | `to:` | `user`, `mode`, `comment` | `COPY --from=<layer-stage> --chmod= [--chown=]` — no RUN |
| `write:` | destination path | `content:` | `user`, `mode`, `comment` | Write inline content — staged + COPY, no shell heredoc |
| `link:` | symlink path (where the link lives) | `target:` | `user`, `comment` | `ln -sf <target> <link>` (coalesces with adjacent) |
| `download:` | URL | — (`to:` unless `extract: sh`) | `user`, `extract`, `to`, `include`, `strip_components`, `mode`, `env`, `comment` | `curl` + optional extract (`tar.gz`/`tar.xz`/`tar.zst`/`zip`/`none`/`sh`). `strip_components: N` emits `tar --strip-components=N` for tar.* — drops leading path segments so tarballs that nest under a top-level arch/version dir (Go, Rust, Node binary releases) land files directly at `to:`. See `/charly-coder:uv` for the canonical uv-x86_64-unknown-linux-gnu/uv → /usr/local/bin/uv example. |
| `setcap:` | file path | — | `user` (implicit root), `caps`, `comment` | File capabilities (`setcap -r` strip if `caps` empty) |
| `build:` | `"all"` | — | `user` (default `${USER}`), `comment` | Run auto-detected builders (pixi/npm/cargo/aur) at this point (instead of end-of-candy) |

### Shared modifiers

| Modifier | Applies to | Default | Purpose |
|---|---|---|---|
| `user` | all verbs | `root` | `root` / `${USER}` / literal username / `<uid>:<gid>` |
| `mode` | `mkdir`, `copy`, `write`, `download` | type-specific (`0755`/`0644`) | Octal permissions |
| `to` | `copy`, `download` | — | Destination in container |
| `target` | `link` | — | What the symlink points to |
| `content` | `write` | — | Inline file body (YAML block scalar) |
| `extract` | `download` | `none` | Archive format — `tar.gz` / `tar.xz` / `tar.zst` / `zip` / `none` / `sh` |
| `include` | `download` | — | Extract only these paths (archive formats) |
| `env` | `download` | — | Env vars for install scripts (`sh` extract) |
| `caps` | `setcap` | empty (= strip) | Capability spec (e.g. `cap_setuid=ep`) |
| `cache` | `cmd`, `download` | — | List of absolute BuildKit cache-mount paths for this task's RUN. Ownership follows `user:` (root → shared/locked, non-root → uid/gid-owned). Persists heavy downloads/build artifacts across builds the SAME way package caches do — surviving an upstream layer cache-miss. The cache-USE logic (sentinel guards, copy-into-place) lives in the task body; charly only mounts the cache. See "Caching downloads" below. |
| `comment` | all | — | Emitted as a Containerfile comment above the task |

### Caching downloads (`download:` auto-caches; `cache:` for heavy installers)

Every `download:` task is **content-addressed cached automatically**: the file
is fetched ONCE into the shared `/tmp/downloads` BuildKit cache mount (keyed by
the URL's sha256), reused across builds, and integrity-safe (curl writes
`<hash>.part`, atomically renamed only on success — a partial/corrupt download,
e.g. from a flaky CDN, is never reused). No author action needed; prefer
`download:` over a hand-rolled `cmd: curl …`.

For a `cmd:` task that downloads/builds via its own tool (an SDK installer, a
source build) and so can't use `download:`, declare a `cache:` mount and route
the tool's downloads into it. Canonical worked example — caching the Android SDK
so an upstream base cache-miss doesn't re-download ~1.5GB from Google's CDN:

```yaml
- cmd: |
    SDK_CACHE=/var/cache/charly-android-sdk
    if [ ! -f "${SDK_CACHE}/.charly-sdk-complete" ]; then
      rm -rf "${SDK_CACHE}"; mkdir -p "${SDK_CACHE}"
      sdkmanager --sdk_root="${SDK_CACHE}" "platform-tools" "emulator" "system-images;…"
      touch "${SDK_CACHE}/.charly-sdk-complete"   # sentinel: only after full success
    fi
    cp -a "${SDK_CACHE}/." /opt/android-sdk/   # materialize into the image
  user: user
  cache:
    - /var/cache/charly-android-sdk                 # owned (uid) because user: user
```

The sentinel (`.charly-sdk-complete`, written only after a fully-successful install)
guards against a partial/interrupted download populating the cache. AUR builds
get the same treatment via `build.yml`'s `aur` builder (owned `cache_mount`
entries for makepkg `SRCDEST` + yay's clone cache) — see `/charly-build:build`.

### Verb examples

```yaml
# Arbitrary shell
- cmd: dnf install -y https://dl.google.com/linux/direct/google-chrome-stable_current_x86_64.rpm
  user: root

# Multi-line shell (shared shell — cd persists)
- cmd: |
    git clone --depth 1 https://github.com/foo/bar /tmp/src
    cd /tmp/src
    cargo build --release
    install -m 755 target/release/bar /usr/local/bin/bar
    rm -rf /tmp/src
  user: root

# Make directory
- mkdir: /etc/traefik/dynamic
  user: root
  mode: "0755"

- mkdir: "${HOME}/.config/sway"
  user: "${USER}"

# Copy existing file from layer dir to container (no RUN, pure COPY)
- copy: traefik.yml
  to: /etc/traefik/traefik.yml
  mode: "0644"
  user: root

# Write new file from inline YAML content (no shell heredoc!)
- write: /etc/X11/xorg.conf
  mode: "0644"
  user: root
  content: |
    Section "ServerFlags"
      Option "DontVTSwitch" "true"
    EndSection

# Create symlink
- link: /usr/local/bin/node
  target: /usr/bin/node-24
  user: root

# Download + extract
- download: "https://github.com/traefik/traefik/releases/download/${TRAEFIK_VERSION}/traefik_${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz"
  extract: tar.gz
  to: /usr/local/bin
  include: [traefik]
  user: root

# Install script piped into shell
- download: "https://astral.sh/uv/install.sh"
  extract: sh
  env:
    UV_INSTALL_DIR: /usr/local/bin
  user: root

# File capabilities
- setcap: /usr/bin/sway        # no caps → strip (setcap -r)

- setcap: /usr/bin/newuidmap
  caps: "cap_setuid=ep"

# Explicit builder placement (lets you run tasks AFTER pixi/npm/cargo)
- build: all
  user: "${USER}"
- cmd: pip install --no-deps ./vllm-nightly.whl
  user: "${USER}"
```

### `copy:` vs `write:` — never conflate

They happen to emit `COPY` directives under the hood, but they have entirely distinct semantics:

| | `copy` | `write` |
|---|---|---|
| Source of bytes | File on disk under `candy/<name>/` | Inline `content:` block in `charly.yml` |
| Replaces old pattern | `cp /ctx/foo bar` + `chmod` | `cat > foo << 'EOF' … EOF` |
| Validation check | `src` must exist under candy dir at generate time | `content` must be non-empty |
| Cache key | Layer-stage file content | Content-addressed staged file at `.build/<image>/_inline/<layer>/<sha256>` |
| Shell-heredoc involvement | None | **None** — content never appears in the Containerfile |

`write:` is the heredoc-safe path: the YAML `content:` is staged to disk at generate time and delivered via `COPY`. That means `$`, backticks, nested `EOF` markers, and arbitrary binary-safe text are all handled without any escaping. If you find yourself writing `cat > foo << 'EOF'` inside a `cmd:` block, use `write:` instead.

---

## `var:` and `${VAR}` Substitution

`var:` is a candy-local `map[string]string`. Values are emitted as `ENV` before the candy's tasks, so every subsequent directive in the candy (including `COPY --chmod=` paths) sees them as shell-resolvable `${VAR}` references.

```yaml
var:
  TRAEFIK_VERSION: v3.4.0

task:
  - download: "https://github.com/traefik/traefik/releases/download/${TRAEFIK_VERSION}/traefik_${TRAEFIK_VERSION}_linux_${ARCH}.tar.gz"
    extract: tar.gz
    to: /usr/local/bin
    include: [traefik]
    user: root
```

### Auto-exports

These names are reserved — `var:` may not shadow them:

| Name | Value | Resolution |
|---|---|---|
| `USER` | image's configured username | Generate-time (from charly.yml resolution) |
| `UID` | numeric user ID | Generate-time |
| `GID` | numeric group ID | Generate-time |
| `HOME` | resolved home directory | Generate-time |
| `ARCH` | BuildKit-style: `amd64` / `arm64` / `ppc64le` / `s390x` | Build-time via `ARG TARGETARCH` + `ENV ARCH=${TARGETARCH}` at candy top |
| `BUILD_ARCH` | uname-style: `x86_64` / `aarch64` / … | Build-time, shell-only (auto-injected inside `cmd:` and `download:` as `BUILD_ARCH=$(uname -m)`) |

**Why two `ARCH` flavours:** `${ARCH}` is BuildKit form because that's what most upstream release naming uses (traefik, mcp-grafana, cosign, etc.). `${BUILD_ARCH}` is uname form because a handful of projects (pixi, yay, just, typst) use `x86_64`/`aarch64` in their release filenames. Pick whichever matches your URL template — the generator handles both.

### Where substitution applies

`${VAR}` is resolved in every task field **except**:

- `cmd:` command text — passed verbatim to bash so shell-style `${VAR:-default}`, `$(command)`, and `$NAME` all work the way you'd expect. **But** RUN shells don't have `$USER` exported by default — use `getent passwd <UID>` for user-name discovery inside `cmd:`, not `$USER`.
- `write: content:` — the file body is verbatim bytes (never substituted, never heredoc'd). If you need the resolved `${USER}` inside a file, use a `cmd:` task that writes via shell redirect after discovering the name via `getent passwd 1000 | cut -d: -f1`. Canonical worked example: `/charly-coder:sshd`'s sudoers drop-in.

Everything else (paths, URLs, modes, `to`, `target`, `user`, etc.) resolves via the Docker-level ENV mechanism, so BuildKit sees the values and substitutes in COPY/RUN/ENV directives.

**Why `$USER` isn't reliable in `cmd:`** — the generator resolves `${USER}` in task fields (paths, `user:` directive, COPY --chown) at generate time. But `cmd:` body text is passed through to bash at RUN time, and bash at RUN time doesn't export `$USER` unless the image explicitly set it via an `ENV USER=` directive (which we don't). `getent passwd <UID>` is the robust, fully-generic alternative that works equally under create mode (`user`) and adopt mode (`ubuntu`) — see `/charly-image:image` "user_policy".

### Validation

- `var:` keys must match `^[A-Z_][A-Z0-9_]*$` (standard shell identifier)
- Keys may not collide with auto-exports or with the candy's own `env:` keys
- Unresolved `${VAR}` in non-shell fields (paths, URLs, etc.) errors at `charly box validate`

---

## Style: Explicit Tasks, No YAML Anchors

Every task — root or user — must be written out in full. Do **not** use YAML anchors (`&name`, `<<: *name`) to share `user:` / `mode:` across tasks. `gopkg.in/yaml.v3` parses anchors correctly, but the repo convention is that every `charly.yml` task reads end-to-end without indirection.

```yaml
# Style across the repo — root and user tasks look identical in shape:
task:
  - mkdir: /etc/traefik
    user: root
    mode: "0755"

  - mkdir: "${HOME}/.local/bin"
    user: "${USER}"

  - copy: chrome-wrapper
    to: "${HOME}/.local/bin/chrome-wrapper"
    mode: "0755"
    user: "${USER}"
  - copy: chrome-restart
    to: "${HOME}/.local/bin/chrome-restart"
    mode: "0755"
    user: "${USER}"
```

Why: anchor merges always override the verb (the interesting field), so the DRY saving is 1–2 trivial fields per task at the cost of forcing readers to jump to the anchor definition to understand each task. Adjacent-coalescing in the generator already collapses identical consecutive operations into one Containerfile directive — there is no build-time cost to explicit repetition.

---

## Execution Order

**Tasks run in the exact YAML order you wrote them. No reordering, ever.** This is load-bearing: `download` before `copy` that references the downloaded path; `mkdir` before `copy` into that directory; `cmd useradd` before `cmd` as that new user.

### Adjacent-coalescing (the only rendering optimisation)

When two or more consecutive tasks share the same verb AND the same resolved `user:` AND the verb supports batching, they render into one Containerfile directive instead of N. This is pure output optimisation — the operations still happen in authored sequence, and no task ever moves past a different-verb or different-user task.

| Verb | Coalesces adjacent same-user? | Output |
|---|---|---|
| `mkdir` | Yes | `RUN mkdir -p p1 p2 p3 …` (one RUN; `chmod` grouped by mode) |
| `link` | Yes | `RUN ln -sf t1 l1 && ln -sf t2 l2 …` |
| `setcap` | Yes | `RUN setcap …` chain |
| `copy` | No coalescing needed | Multiple `COPY` directives back-to-back, no RUN between them |
| `write` | No coalescing needed | Same as `copy` but from staged-inline content |
| `download` | No | Each URL gets its own RUN — merging would hide failures |
| `cmd` | No | Each `cmd:` maps 1:1 to one RUN — merging arbitrary shell would erase author intent |
| `build` | Singleton | One per candy (auto or explicit) |

**Non-adjacent same-verb tasks never coalesce.** `mkdir /a` → `copy foo /a/foo` → `mkdir /b` renders as three directives, not two. Collapsing the two mkdirs would change observable state (the copy would see a different directory layout) and is forbidden.

### `mkdir -p` ancestor registration

When you declare `mkdir: /a/b/c`, the generator also treats `/a` and `/a/b` as "declared" — because Unix `mkdir -p` creates the full path. That means a later `copy` into `/a/b/foo` won't trigger a redundant auto-inserted `mkdir /a/b`. This matches author intent: you wrote one mkdir, you get one mkdir, even if multiple COPYs land under the same tree.

### Parent-directory auto-insertion

The single, tightly-scoped exception to "no implicit inserts": if a `copy:` or `write:` task's destination parent isn't already declared (directly or via ancestor-registration above), the generator prepends a single `RUN mkdir -p <parent>` (as the task's `user:`) **immediately before** the COPY — never relocating anything, never merging across other tasks. To opt out of auto-insertion for a specific path, declare the parent explicitly with `mkdir:` earlier in the list.

### USER directives

The generator tracks the running USER and emits `USER <value>` only when the next task's resolved user differs. Eight consecutive `${USER}` tasks get one `USER 1000` directive followed by eight directives — no redundant switches. Between different-user tasks, a `USER` switch happens where author order requires.

---

## User Resolution

The `user:` field accepts:

| Value | Emits | COPY `--chown` pair |
|---|---|---|
| `root` / `"0"` / omitted | `USER 0` | — (root is the COPY default) |
| `${USER}` | `USER <numeric UID>` | `<UID>:<GID>` (e.g. `1000:1000`) |
| `<uid>:<gid>` (e.g. `1010:1010`) | `USER <uid>:<gid>` | same |
| bare numeric `<uid>` | `USER <uid>` | `<uid>:<uid>` |
| literal name (`postgres`, `immich`) | `USER <name>` | `<name>:<name>` |

**Why `${USER}` resolves to a numeric UID:** avoids a `/etc/passwd` dependency at the instant the switch happens. Base images create the named user, but numeric always works even if `/etc/passwd` hasn't been read yet by whatever shell the RUN spawns.

**User creation is explicit.** If you reference a literal username, an earlier `cmd:` task (as root) must `useradd` them. The generator does not auto-create users. Example:

```yaml
task:
  - cmd: |
      useradd -r -u 1010 -g root -s /sbin/nologin immich
      mkdir -p /srv/immich && chown -R immich:root /srv/immich
    user: root
  - cmd: |
      cd /srv/immich
      pnpm install --frozen-lockfile
      pnpm build
    user: immich
```

---

## charly.yml Field Reference

| Field | Type | Purpose |
|-------|------|---------|
| `version` | `string` | **MANDATORY** CalVer (`YYYY.DDD.HHMM`) of this candy definition — the candy kind requires it (`charly box validate` hard-errors when absent; `charly migrate` backfills it). The authoritative per-entity identity: it drives cross-repo layer resolution (`pickCandyVersion`) and, as the highest layer version, the consuming image's content-stable `ai.opencharly.version` label. Bump it when the candy's content changes. |
| `status` | `string` | `working`, `testing`, or `broken`. Default: `testing`. |
| `info` | `string` | Free-form description of what works / doesn't. Recommended for `testing` / `broken`. |
| `require` | `[]string` | Candy dependencies. Resolved transitively, topologically sorted. |
| `layer` | `[]string` | Compose other candies into this one (splicing). |
| `env` | `map[string]string` | Container-runtime environment variables. Merged across candies. |
| `path_append` | `[]string` | Paths appended to `$PATH`. Deduplicated. |
| `var` | `map[string]string` | **Build-time** candy-local variables for `${VAR}` substitution. Emitted as `ENV` before tasks. |
| `task` | `[]Task` | **Ordered** install operations. See Task Verb Catalog above. |
| `ports` | `[]int \| []PortSpec` | Exposed ports (1-65535). Protocol-annotated form: `tcp:5900`, `https+insecure:3000`, etc. |
| `route` | `{host, port}` | Traefik reverse proxy route. |
| `service` | multiline string | Supervisord `[program:<name>]` fragment. |
| `package` / `distro` | list / map | The package surface: top-level `package:` base + per-distro/version `distro:` map (bare / versioned / compound keys). Resolved via the most-specific-first cascade (see Package Surface). |
| `apk` | `[]ApkPackageSpec` | **Android app-install package format** — apps installed onto a `kind: android` device by a `target: android` deploy (NOT into the image). Each entry is `package:` (apkeep download by id, with `source`/`arch`/`version`) XOR `apk:` (committed local APK). Device-scoped (top-level, not under `distro:`); compiles to an `ApkInstallStep` that ONLY `target: android` executes (skipped at image-build + on every other target). See `/charly-eval:android`. |
| `localpkg` | map (format→dir) | **Native-package deploy format** (sibling of `apk`). A per-format map (`pac`/`rpm`/`deb` → a bundled package SOURCE dir, e.g. the `charly` layer's `{pac: pkg/arch, rpm: pkg/fedora, deb: pkg/debian}`). Compiles to a `LocalPkgInstallStep` (`/charly-internals:install-plan`) emitted at "step 2.5", BEFORE the layer's `task:`. On a DEPLOY target (`target: local` / `target: vm`) charly picks the entry matching the target distro's package format, builds the package on the HOST (pac via `makepkg`; rpm/deb distro-natively in a podman container), and installs it via the format's AUTO-RESOLVING local-file install (`pacman -U` / `dnf install` / `apt-get install`) — so the package's repo dependencies resolve automatically and there is NO dependency-closure builder. Every command (build / install / probe / glob / `source_sentinel`) is rendered from the distro's `format.<fmt>.local_pkg:` block in `build.yml` — zero hardcoded package-manager literals in Go. The legacy scalar form (`localpkg: pkg/arch`) is rejected at load with an `charly migrate` hint. Resolution walks up from the deploy project dir, so a consumer under `box/<distro>` finds the superproject's `pkg/<fmt>`. Skipped at image build — the layer's own `task:` (curl/COPY) is the fallback. The `charly` layer uses this to install `charly` as the native OS package at `/usr/bin/charly` on every distro instead of a curl'd binary. |
| `volumes` | `[]{name, path}` | Persistent named volumes. |
| `aliases` | `[]{name, command}` | Host command aliases. |
| `security` | object | Container security: `privileged`, `cap_add`, `devices`, `security_opt`, `shm_size`, resource caps. |
| `port_relay` | `[]int` | Ports needing eth0 → loopback socat relay. Auto-adds `socat` dependency. |
| `secrets` | `[]SecretYAML` | Image-owned container secrets (auto-generated per instance). |
| `hooks` | `HooksConfig` | Lifecycle hooks: `post_enable` (after `charly config`), `pre_remove` (before `charly remove`). |
| `libvirt` | `[]string` | Raw libvirt XML snippets for VM domain XML injection. |
| `data` | `[]DataYAML` | Data mappings (`src` → volume `dest`) for volume staging. |
| `env_provide` | `map[string]string` | Env vars injected into OTHER containers when this service is deployed. Template: `{{.ContainerName}}`. |
| `env_require` | `[]EnvDependency` | Plaintext env vars this candy MUST have. Hard error at `charly config` if missing. |
| `env_accept` | `[]EnvDependency` | Plaintext env vars this candy CAN optionally use. Opt-in allowlist for `env_provide` injection. |
| `secret_accept` / `secret_require` | `[]EnvDependency` | Credential-backed env vars. Values live in credential store, never in charly.yml/quadlet. |
| `mcp_provide` / `mcp_require` / `mcp_accept` | various | MCP server discovery analogous to `env_*`. |
| `service` | `[]ServiceEntry` | Unified service list — see "Service Declaration" below. |

Field details for non-`task:` sections are below.

---

## Package Surface — the `distro:` map (single canonical surface)

Packages are declared ONLY under the `distro:` map, plus an optional top-level
`package:` base. There is NO top-level `rpm:` / `deb:` / `pac:` package-format
key and NO top-level `debian:13:` / `debian,ubuntu:` tag key — those legacy forms
are removed (a stray one is a hard load error pointing at `charly migrate`, which the
existing `calamares` step rewrites into the `distro:` map). The arch `aur:`
sub-block is the one exception that stays a build format — see AUR below.

```yaml
package:                 # the always-included BASE — installed on EVERY distro
  - git                  #   (top-level list of cross-distro-valid package names)

distro:
  fedora:                # bare distro name → applies to every fedora version
    package: [vim, ripgrep]
    copr: [owner/project]
    option: ["--setopt=tsflags=noscripts"]
  arch:
    package: [vim, ripgrep]
  "debian,ubuntu":       # COMPOUND → shared by both (split into one tag each)
    package: [vim, ripgrep]
    repo:                # /etc/apt/sources.list.d/<name>.list + keyring
      - name: myrepo
        url: "https://example.com/repo"
        key: "https://example.com/key.gpg"   # ASCII-armored; gpg --dearmor'd
        suite: stable
        components: main
  ubuntu-24.04:          # VERSIONED (dash form) → overrides for that version only
    repo: [{name: nodesource, url: "...", suite: nodistro, components: main}]
```

### Key forms (all under `distro:`)

| Form | Example | Means |
|---|---|---|
| bare distro | `fedora` / `debian` / `ubuntu` / `arch` | every version of that distro |
| versioned | `debian-13` / `ubuntu-24.04` | that specific version (dash form — robust YAML) |
| compound | `"debian,ubuntu"` / `"debian-13,ubuntu-24.04"` | shared across the listed distros/versions (split into one per distro) |

Each carries the full surface: `package` + `repo` + `copr` + `option` + `exclude`
+ `module` (and `aur` under `arch`). `repo` is the canonical key (singular).

### Resolution is a most-specific-first CASCADE

For an image, the resolver walks the image's `distro:` tag chain — most-specific
first (e.g. `[ubuntu:24.04, ubuntu]` or `[debian:13, debian]`) — plus the
top-level `package:` base:

- **packages = UNION** across the base + every matching level (deduped). A shared
  package on a broad level plus extras on a specific level accumulate.
- **repo / copr / option / exclude / module = MOST-SPECIFIC matching level wins**
  (a `ubuntu-24.04` repo overrides the bare `ubuntu` repo).

This is what makes per-distro repos DETERMINISTIC: each distro owns its own tag
section, so debian and ubuntu can never race over one shared section (the old
collapse that made the apt suite — trixie vs noble — depend on Go map order).
Canonical examples: `/charly-tools:charly` (tailscale `debian`→trixie / `ubuntu`→noble),
`/charly-coder:docker-ce` (per-version repos), `/charly-tools:gh` (compound shared repo).

### The cascade cannot SUBTRACT — express exclusions structurally

Because packages UNION across the chain, you cannot have a specific level REMOVE a
package a broader level added. To keep a package OFF a distro/version, simply
never list it on any level that distro resolves. Worked example: `/charly-coder:dev-tools`
keeps `fastfetch` (absent from Ubuntu's repos) only under `debian` + `fedora` +
`arch` — NOT under `ubuntu` — so the ubuntu cascade never includes it.

### Image distro chains (where the most-specific-first order comes from)

An image declares its chain in `charly.yml` `distro:` (e.g. `[debian:13, debian]`,
`[ubuntu:24.04, ubuntu]`); a `target: vm` deploy synthesizes the SAME chain from
the distro's canonical `version:` in `build.yml` via `distroTagChain` — so
per-version selection works identically on image builds and VM deploys. fedora /
arch carry only a bare tag (`[fedora]` / `[arch]`) and reach their packages via
the bare-distro tag section. See `/charly-build:build` and `/charly-internals:install-plan`.

### Pac (`pac:`)

```yaml
pac:
  packages:
    - neovim
    - ripgrep
  repos:
    - name: custom-repo
      server: https://example.com/repo/$arch
      siglevel: Optional TrustAll    # default if omitted
  options:
    - --needed
```

### AUR (`aur:`)

AUR packages are compiled by `yay` in a multi-stage build, then the resulting
`.pkg.tar.zst` artifacts are `pacman -U`-installed into the final image. Two
conditions must BOTH hold for the AUR stage to be emitted:

1. The consuming **image declares `aur` in its `build:` list** (e.g.
   `build: [pac, aur]`). The `arch` AND `cachyos` bases both declare only
   `build: [pac]`, so a consumer that needs AUR MUST add `aur` itself (the
   `arch-test` and `selkies-desktop` precedents). Without it the AUR section is
   silently skipped — by design, so a multi-distro candy can carry `aur:` for
   Arch consumers without forcing every Fedora consumer to invoke a builder.
2. **`builder.aur` is configured** — `arch-builder` for BOTH the `arch` and
   `cachyos` bases. CachyOS is Arch-derived and routes all four
   pixi/npm/cargo/aur formats to `arch-builder`, so an `aur:` layer builds
   **identically on arch and cachyos** — there is no cachyos-specific AUR path.

**Canonical form — nested under `distro.arch`** (how the repo's candies author it:
`chrome`, `vscode`, `wl-tools`). The `aur:` block sits beside the regular
`package:` list inside the arch distro section, so ONE candy carries Fedora RPMs
and Arch repo + AUR packages together; the generator picks the section matching
the image's `distro:` tags:

```yaml
distro:
  arch:
    package:                 # Arch repo (core/extra) packages
      - vulkan-icd-loader
    aur:                     # AUR packages — need build: [pac, aur] on the image
      package:
        - google-chrome
      options:
        - --nocheck
      # `replaces:` lists distro-repo packages whose file paths conflict
      # with the AUR build artifact. Each entry is removed via
      # `pacman -Rs --noconfirm <pkg>` BEFORE the AUR `pacman -U` install.
      # Idempotent — entries that aren't installed are silently skipped.
      # Required when the AUR build owns paths also owned by an Arch repo
      # package (e.g. `visual-studio-code-bin` and `code` both own /usr/bin/code).
      replaces:
        - code
  fedora:
    package:
      - vulkan-loader
```

A top-level `aur:` block (sibling of `pac:`, with a plural `packages:` list) is
the single-distro shorthand for an Arch-only candy; both forms flatten to the
same internal `aur` format section (`charly/layers.go` `derivePackageSectionsFromCalamares`).
Prefer the nested `distro.arch.aur.package` form for multi-distro candies.

The `replaces:` mechanism applies to host (`target: local`) deploys; in OCI image builds the candy is applied to a fresh rootfs that never has the conflicting package, so `replaces:` is a no-op there.

---

## Dependencies

Candies declare dependencies via `require`. The generator resolves transitively, topologically sorts, and pulls missing dependencies automatically. Circular dependencies are a validation error.

```yaml
require:
  - python
  - supervisord
```

### `require` vs `layer`

| | `require` | `layer` |
|---|---|---|
| Purpose | Prerequisite ordering | Group composition |
| Effect | Ensures dependency is installed first | Splices candies at this candy's position |
| Transitive | Yes — pulls in sub-dependencies | Yes — recursively expands |
| Typical use | Runtime/build prerequisites | Metalayers, candy bundles |

**Common mistake:** `require: [pixi]` when you mean `require: [python]`. The `pixi` candy installs the pixi binary (build tool). The `python` candy installs Python via pixi. Your candy needs Python.

---

## Environment Variables

```yaml
env:
  PIXI_CACHE_DIR: "~/.cache/pixi"
  MY_VAR: "value"

path_append:
  - "~/.pixi/bin"
  - "~/.local/bin"
```

`~` and `$HOME` expand to the resolved home directory at generation time. Setting `PATH` directly in `env` is a validation error — use `path_append`. Later candies override earlier for the same key.

**`env:` vs `var:`:** `env:` is container **runtime** environment (emitted as `ENV` and persists into the running container). `var:` is **build-time** substitution for `${VAR}` references inside `task:` — also emitted as `ENV` so BuildKit can substitute in COPY paths, but conceptually scoped to the candy's install. There's no hard rule against using `env:` for both purposes, but keeping them separate makes intent clearer.

**`env:` is a MAP, not a list.** The YAML parser decodes it as `map[string]string`, not `[]string`. Authoring it as `- KEY=value` fails with `cannot unmarshal !!seq into map[string]string` at `charly box validate`. Always use map form:

```yaml
# ❌ WRONG — parser rejects the list shape
env:
  - CHARLY_PROJECT_DIR=/workspace
  - GOPATH=~/go

# ✓ RIGHT — map form
env:
  CHARLY_PROJECT_DIR: "/workspace"
  GOPATH: "~/go"
```

The `charly-mcp` candy is the canonical example of the map form used to thread a container-level env var into the MCP server process via supervisord.

---

## Service Declaration — unified `service:` schema

A candy declares services via the **unified `service:` list**. One schema covers both distro-packaged systemd units and fully custom entries, rendered to supervisord INI (for container init) or systemd unit files (for bootc images + host deploys) by the init-system's `service_schema` in `build.yml`.

Every entry has one `name:` plus either a `use_packaged:` reference (reuse a distro-shipped unit) OR a structured custom spec (`exec`, `env`, `restart`, ...). The two forms are mutually exclusive.

### Form 1 — reuse a packaged systemd unit

For services shipped by a distro package (postgresql, nginx, redis, sshd, ...). charly enables the packaged unit with optional drop-in overrides — it never regenerates the unit file. The packaged unit at `/usr/lib/systemd/system/<unit>.service` stays untouched; override config lands at `/etc/systemd/system/<unit>.service.d/charly-<layer>.conf`.

```yaml
service:
  - name: postgresql
    use_packaged: postgresql.service     # distro-shipped unit
    enable: true                          # systemctl enable --now
    scope: system                         # system | user
    overrides:                            # optional — rendered as drop-in
      env:
        PGDATA: /var/lib/postgresql/data
      after: [network-online.target]
```

**Supervisord caveat**: `use_packaged:` is a systemd-only concept. On supervisord-targeted images (containers using the default init), packaged entries render a warning and get skipped; either author a custom entry (form 2) or target a systemd image.

### Form 2 — custom service

For services that aren't distro-packaged (ollama, custom daemons, candy-provided binaries). charly renders the spec through the init-system's `service_template` in `build.yml` — supervisord-init containers get INI fragments, systemd-init containers and bootc/host deploys get `.service` unit files.

```yaml
service:
  - name: ollama
    exec: /usr/bin/ollama serve
    env:
      OLLAMA_HOST: "0.0.0.0:11434"
      OLLAMA_KEEP_ALIVE: "-1"
    restart: always                       # no | on-failure | always | unless-stopped
    working_directory: /var/lib/ollama
    user: ollama
    after: [network.target]
    before: [ollama-exporter]
    stdout: journal                       # journal | file:<path> | none
    stop_timeout: "30s"
    scope: system                         # system | user
    enable: true
```

### Mixed entries in one candy

A candy can declare multiple entries mixing both forms. The `sshd` candy is the canonical example: it enables the packaged `sshd.service` for systemd-init scope AND runs a custom wrapper via supervisord.

```yaml
service:
  - name: sshd
    use_packaged: sshd.service           # bootc/systemd scope
    enable: true
    scope: system

  - name: sshd                            # name may repeat across forms (different init system renders each)
    exec: /usr/local/bin/sshd-wrapper
    restart: always
    priority: 3
    enable: true
    scope: system
```

### Field reference

| Field | Required | Applies to | Description |
|---|---|---|---|
| `name` | yes | both | Service identifier; passed to `service_template` as `.Name` |
| `use_packaged` | form 1 | packaged | Distro-shipped unit name (e.g., `postgresql.service`); mutually exclusive with `exec` |
| `exec` | form 2 | custom | Command to run (`ExecStart` in systemd, `command` in supervisord) |
| `env` | no | both | Map of env vars; systemd renders as `Environment="K=V"`, supervisord as `environment=K="V",...` |
| `restart` | no | custom | `no` / `on-failure` / `always` / `unless-stopped`; mapped by init-specific template helper |
| `working_directory` | no | custom | `WorkingDirectory=` (systemd) / `directory=` (supervisord) |
| `user` | no | custom | Service runtime user |
| `after` | no | both | List of unit names this service starts after; systemd `After=`, supervisord priority ordering |
| `before` | no | custom | Inverse of `after` |
| `stdout` | no | custom | `journal` / `file:<path>` / `none`; init-specific mapping |
| `stop_timeout` | no | custom | `TimeoutStopSec=` (systemd) / `stopwaitsecs=` (supervisord) |
| `scope` | no | both | `system` (default, `/etc/systemd/system/`) or `user` (`~/.config/systemd/user/`) |
| `enable` | no | both | Whether to `systemctl enable --now` at install time |
| `overrides` | no | packaged | Drop-in modifications: `env`, `after`, `exec` |
| `priority` | no | supervisord-only | Startup priority; translated from `after`/`before` on systemd |

### Lifecycle-directive overlay (supervisord-only fields)

Beyond the core `service:` schema above, supervisord-rendered entries accept additional lifecycle fields: `auto_start`, `start_retries`, `start_secs`, `stop_signal`, `exit_codes`, `priority`, `kind: eventlistener` + `events:`. See `/charly-infrastructure:supervisord` for the directive table and `/charly-selkies:chrome` for the eventlistener worked example.

### Rendering to init systems

The actual unit text is rendered by the init-system's `service_schema` block in `build.yml`:

- **Supervisord init** — `service_template` produces `[program:NAME]` INI fragments with `autorestart` / `environment` / etc.; fragments go to `/etc/supervisord.d/<layer>-<name>.conf` and are assembled at container-build time.
- **Systemd init (bootc + host deploys)** — `service_template` produces `[Unit]` / `[Service]` / `[Install]` blocks; the rendered file goes to `/etc/systemd/system/charly-<layer>-<name>.service` (or the user-scope path when `scope: user`). For `use_packaged:` entries, `dropin_template` + `dropin_path_template` produce an override file alongside the packaged unit.

See `/charly-infrastructure:supervisord` for the supervisord ServiceSchemaDef template, `/charly-build:build` for the three-phase template model, and `/charly-local:local-deploy` for how the host target consumes `service:` entries.

### Worked examples in-tree

- `/charly-infrastructure:postgresql` — canonical `use_packaged:` entry with drop-in overrides
- `/charly-ollama:ollama` — single custom entry (common shape)
- `/charly-hermes:hermes` — custom entry with complex env and ordering
- `/charly-coder:sshd` — mixed (packaged + custom) within one candy
- `/charly-infrastructure:virtualization` — mixed entries for virtqemud/virtnetworkd (canonical polymorphism example)
- `/charly-infrastructure:traefik` — multiple custom entries for a multi-service candy

### Anti-pattern: `<name>-host` / `<name>-pod` sibling candies

Do **NOT** split a polymorphic service into two candies like `myservice` (supervisord variant) + `myservice-host` (systemd variant). The mixed-entries pattern above (same `name:`, one `use_packaged:` entry, one `exec:` entry — init system at deploy time picks) is the supported way to carry both.

If you find yourself reaching for a `-host` suffix on a candy name, reach for a second `service:` entry instead. The same rule applies to `-pod` suffixes for the inverse case (a candy that needs container-only behavior under systemd targets).

`charly box validate` does NOT (yet) statically reject `*-host` / `*-pod` candy names, because some legitimate uses might exist (a candy whose package literally only exists on host distros). The rule lives in CLAUDE.md "Init-system polymorphism via mixed `service:` entries" + this skill + `/charly-infrastructure:supervisord` — agents that load any of those before authoring will see the guidance. Canonical worked examples: `/charly-coder:sshd` (mixed), `/charly-infrastructure:virtualization` (mixed; CANONICAL example), `/charly-infrastructure:postgresql` (use_packaged-only).

## Volume Declaration

```yaml
volume:
  - name: data
    path: "~/.myapp"
```

Names must match `^[a-z0-9]+(-[a-z0-9]+)*$`. Docker/podman volume names become `charly-<image>-<name>`. Collected across the full image base chain; first declaration wins.

## Security Declaration

```yaml
security:
  privileged: false
  cap_add:
    - SYS_PTRACE
  devices:
    - /dev/dri
  security_opt:
    - label:disable
  group_add:
    - keep-groups
  mounts:
    - /dev/input:/dev/input:rw
    - tmpfs:/run/udev:rw,size=1m
  shm_size: "1g"
  memory_max: "8g"           # hard limit
  memory_high: "6g"          # soft limit
  memory_swap_max: "0"       # disable swap
  cpus: "4.0"                # CPU quota
```

Security settings merge across candies (union for lists; `privileged` true if any candy sets it; smallest-wins for resource caps). Box-level `security:` in `charly.yml` overrides `privileged` and replaces resource caps.

Resource caps (memory / cpus) bound the blast radius of a Chrome crash loop on the chrome candy. See `/charly-selkies:chrome` (Resource Caps) and `/charly-infrastructure:supervisord` for the (generic) eventlistener pattern.

---

## port_relay

Ports needing an eth0 → loopback socat relay inside the container. For services that bind only to 127.0.0.1. Auto-adds `socat` dependency and generates a relay service.

```yaml
port_relay:
  - 9222
```

Note: the chrome candy uses a dedicated `cdp-proxy` supervisord service instead of `port_relay`, to handle Chrome 146+ Host header validation.

## Port Protocol Annotations

Ports support a protocol prefix that controls tunnel backend scheme and EXPOSE format:

```yaml
port:
  - 18789                   # http (default) → tailscale --https, cloudflared http://
  - "https+insecure:3000"   # https+insecure → HTTPS tunnel with self-signed OK
  - tcp:5900                # raw TCP
  - "tls-terminated-tcp:22" # tailscale TLS-terminated-tcp
  - udp:47998               # EXPOSE /udp, not tunneled
  - 9222                    # http (default)
```

Tailscale schemes: `http`, `https`, `https+insecure`, `tcp`, `tls-terminated-tcp`. Cloudflare adds `ssh`, `rdp`, `smb`. HTTPS backends (Traefik with self-signed certs) MUST use `https+insecure` — plain `http` proxying to HTTPS returns 404.

---

## secrets (image-owned)

```yaml
secret:
  - name: api-key                # Podman secret `charly-<image>-<name>`
    target: /run/secrets/api_key # mount path (default: /run/secrets/<name>)
    env: API_KEY                 # fallback env var if Podman secrets unavailable
```

Metadata only lives in OCI labels. Values are auto-generated per instance at `charly config` time. Use for image-internal secrets (like `db-password`). For user-owned credentials (API keys, auth tokens), use `secret_accept` / `secret_require` instead.

---

## env_provide / env_require / env_accept

Cross-container environment-variable service discovery. `env_provide` is the supply side; `env_require` (mandatory) and `env_accept` (opt-in) are the demand side.

```yaml
env_provide:
  OLLAMA_HOST: "http://{{.ContainerName}}:11434"
  PGHOST: "{{.ContainerName}}"
  PGPORT: "5432"

env_require:
  - name: DATABASE_URL
    description: "Postgres connection URL"
    default: "postgres://localhost:5432/app"

env_accept:
  - name: HTTP_PROXY
    description: "Upstream HTTP proxy (optional)"
```

`{{.ContainerName}}` resolves at `charly config` time. `env_provide` values are injected only into consumers that declare matching `env_accept` or `env_require` (opt-in filtering — prevents env var leakage). Missing `env_require` without a default is a hard error at `charly config`; missing `env_accept` silently drops the var.

See `/charly-core:charly-config` (`--update-all` flag, provides filtering) and `/charly-core:deploy` (charly.yml `provides:` section) for the full lifecycle.

## secret_accept / secret_require

Credential-backed env vars. Same YAML shape as `env_accept` / `env_require`, but values flow through the credential store (keyring → config) and arrive via Podman secrets — **never plaintext in charly.yml or the quadlet**.

```yaml
secret_require:
  - name: WEBUI_ADMIN_PASSWORD
    description: "Initial admin account password"

secret_accept:
  - name: OPENROUTER_API_KEY
    description: "API key for OpenRouter LLM inference"
    key: charly/api-key/openrouter     # optional override; default charly/secret/<NAME>
```

Use for API keys, passwords, auth tokens. `key:` override must match `^charly/<service>/<key>$` (lowercase). Multiple candies sharing the same upstream credential (e.g. `charly/api-key/openrouter`) all resolve to the same stored value. See `/charly-build:secrets` for the credential-store chain, rotation, and `-e NAME=VAL` auto-import.

## mcp_provide / mcp_require / mcp_accept

Cross-container MCP server discovery. Consumers receive `CHARLY_MCP_SERVERS` as a JSON env var at `charly config` time.

```yaml
mcp_provide:
  - name: jupyter
    url: "http://{{.ContainerName}}:8888/mcp"
    transport: http                # or "sse"

mcp_accept:
  - name: jupyter
    description: "JupyterLab CRDT MCP server for notebook manipulation"
```

**Pod-aware:** when provider and consumer share a container, URLs resolve to `localhost` (local wins over remote for same-named entries). **Naming is the service contract** — keep `name:` stable across candy/package/box renames.

**Testing the endpoint:** once a candy is deployed, `charly eval mcp ping <image>` verifies the server is alive, and `charly eval mcp list-tools <image>` enumerates the tool catalog. Both are authorable as deploy-scope `mcp:` declarative checks inside the candy's `eval:` block. The full verb reference (methods, URL rewriting, port-publishing gotcha, validator rules) lives in `/charly-build:charly-mcp-cmd`.

---

## data

Data candies stage files from the candy directory into volume bind-mount areas. Build-time: files COPY into `/data/<volume>/[dest/]`. Deploy-time: `charly config --bind <volume>` provisions them into bind directories; `charly update` merges non-destructively.

```yaml
volume:
  - name: workspace
    path: "~/workspace"

data:
  - src: data/notebooks
    volume: workspace
    dest: ""                 # optional subdirectory within volume
```

**Data candies** are candies with only `data:` + `volume:` — no packages, no services, no tasks. Valid standalone. **Data images** (`data_image: true` in charly.yml) are scratch-based — consumed via `charly config --data-from <image>`. See `/charly-jupyter:notebook-templates` for a worked example.

---

## Cache Mounts (used by the generator)

The generator attaches cache mounts automatically based on task context:

| Emission site | Cache path | Options |
|---|---|---|
| `rpm.packages` (dnf install) | `/var/cache/libdnf5` | `sharing=locked` |
| `deb.packages` (apt install) | `/var/cache/apt` + `/var/lib/apt` | `sharing=locked` |
| `pac.packages` + `aur` | `/var/cache/pacman/pkg` | `sharing=locked` |
| `cmd:` as root | distro-format caches (as above) + `/ctx` bind to layer stage | — |
| `cmd:` as non-root | `/tmp/npm-cache` (uid-scoped) + `/ctx` bind | `uid=<UID>,gid=<GID>` |
| `download:` | `/tmp/downloads` (shared across layers) | — |
| pixi builder stage | `/tmp/pixi-cache` + `/tmp/rattler-cache` | `uid=<UID>,gid=<GID>` |
| npm builder stage | `/tmp/npm-cache` | `uid=<UID>,gid=<GID>` |
| cargo inline | `/tmp/cargo-cache` | `uid=<UID>,gid=<GID>` |

UID/GID in non-root cache mounts are dynamic (from resolved image config). Flat `/tmp/<tool>-cache` paths avoid buildah permission issues with nested paths.

---

## Common Workflows

### Create a new candy

```bash
charly box new candy my-tool
# Edit candy/my-tool/charly.yml — add packages, deps, env, tasks
charly box validate
```

### Add system packages

Add a `distro:` section to `charly.yml` keyed by distro name (`distro.fedora`, `distro.debian`, `"debian,ubuntu"` compound, `debian-13` versioned), and/or a top-level `package:` base for cross-distro names. The resolver cascades most-specific-first (packages union, repo/options most-specific-wins). `charly candy add-rpm`/`add-deb`/`add-pac`/`add-aur` append into the right `distro:` section for you.

### Add Python packages

Create `pixi.toml` in the candy directory. **`charly.yml` must depend on `python`, not `pixi`** (the `pixi` candy installs the pixi binary; the `python` candy installs Python via pixi). Never use `pip install`, `conda install`, `pixi global install`, or `uv tool install` inside a `cmd:` task. Always use `pixi.toml`.

**In-tree Python packages** shipped by a candy live at `candy/<layer-name>/<pkg-name>/<pkg-name>/` with `pyproject.toml` at the distribution root. Internal imports must be relative (`from .app import X`) so the package directory can be renamed without editing every `.py` file.

### Add npm packages

Create `package.json` in the candy directory; `charly.yml` depends on `nodejs`. The build system runs `npm install -g` in a multi-stage build. Do **not** put `npm install -g` inside a `cmd:` task — `package.json` is the declarative path.

### Add Go packages

Go has no declarative manifest for global installs, so use `cmd:`:

```yaml
require:
  - golang

env:
  GOPATH: "~/go"

path_append:
  - "~/go/bin"

task:
  - cmd: |
      go install github.com/org/tool/cmd/tool@latest
      go clean -cache
    user: "${USER}"
```

For cgo dependencies, add the required `-devel` packages to `rpm:`. Always `go clean -cache` to shrink the image.

### Add a binary download

```yaml
var:
  TOOL_VERSION: v1.2.3

task:
  - download: "https://github.com/org/tool/releases/download/${TOOL_VERSION}/tool-linux-${ARCH}.tar.gz"
    extract: tar.gz
    to: /usr/local/bin
    include: [tool]
    user: root
```

Use `${ARCH}` (BuildKit-style) if the release URL uses `amd64`/`arm64`; use `${BUILD_ARCH}` if it uses `x86_64`/`aarch64`. If the project only ships x86_64, omit the template and hardcode — don't fake multi-arch.

### Install a wrapper script + make it the default

```yaml
task:
  - mkdir: "${HOME}/.local/bin"
    user: "${USER}"
  - copy: my-wrapper
    to: "${HOME}/.local/bin/my-wrapper"
    mode: "0755"
    user: "${USER}"
  - link: "${HOME}/.local/bin/my-tool"     # make my-tool always go through the wrapper
    target: my-wrapper
    user: "${USER}"
```

### Write a config file inline

Use `write:` — never shell heredoc:

```yaml
task:
  - write: /etc/my-app/config.yml
    mode: "0644"
    user: root
    content: |
      listen: 0.0.0.0:8080
      log_level: info
      backends:
        - http://localhost:9090
```

### Add a service

Declare `service:` with a supervisord `[program:<name>]` fragment and add `supervisord` to `require:`. The generator assembles per-candy service fragments into a single `/etc/supervisord.conf` at image build time.

---

## Style Guide

- Lowercase-hyphenated names for candies.
- System packages in the `charly.yml` `distro:` map (+ top-level `package:` base) — not in `cmd:`.
- Python in `pixi.toml`, npm in `package.json`, Rust in `Cargo.toml`. Never `pip install` / `conda install` / `dnf install python3-*`.
- Binary downloads via `download:` verb; use `${ARCH}` or `${BUILD_ARCH}` for multi-arch URL templates.
- Never `dnf clean all` / `pacman -Scc` inside a `cmd:` — cache mounts handle it.
- Prefer `mkdir:` + `copy:` + `link:` + `write:` over `cmd:`. Fall back to `cmd:` only for true escape-hatch logic (git clone + cargo build, complex conditionals, etc.).
- Tasks are order-sensitive — put `download:` / `mkdir:` before the `copy:` / `write:` / `cmd:` that depends on them.
- No YAML anchors: write every task explicitly (same shape for `user: root` and `user: "${USER}"`).

---

## Shell Init Surface (`shell:`)

Candies declare per-shell init snippets via the structured `shell:` field.
Same body shape as per-distro `rpm:`/`pac:`/`deb:`:
intrinsic fields apply to every shell, optional sub-blocks named after a
shell allowlist key (`bash` / `zsh` / `fish` / `sh`) override the
intrinsic for that one shell.

```yaml
# charly.yml — generic + override (the canonical shape)
shell:
  init: |
    # Applies to bash, zsh, sh — ${SHELL_NAME} substituted at install time.
    eval "$(direnv hook ${SHELL_NAME})"
  fish:
    init: |
      # Different syntax — explicit override.
      direnv hook fish | source

# Pure per-shell form
shell:
  bash: { init: 'eval "$(direnv hook bash)"' }
  zsh:  { init: 'eval "$(direnv hook zsh)"' }
  fish: { init: 'direnv hook fish | source' }

# PATH contributions (rendered with shell-appropriate syntax —
# fish_add_path for fish, export PATH= for bash/zsh/sh).
shell:
  path_append:
    - "~/.pixi/bin"
```

**Selection rule** — applied per (target, shell) at install time:
1. If `shell.<shell>.init` exists, use it verbatim.
2. Otherwise, `shell.init` with `${SHELL_NAME}` substituted.
3. Otherwise, the candy contributes nothing for that shell.

**Where snippets land** — destination is target-aware:

| Shell | Container image (`charly box build`) | `target: local` host / `target: vm` guest |
|---|---|---|
| bash | `/etc/profile.d/charly-<layer>-bash.sh` | managed-block in `~/.bashrc` |
| zsh  | `/etc/profile.d/charly-<layer>-zsh.sh`  | managed-block in `~/.zshrc`  |
| sh   | `/etc/profile.d/charly-<layer>-sh.sh`   | managed-block in `~/.profile` |
| fish | `/etc/fish/conf.d/charly-<layer>.fish`  | `~/.config/fish/conf.d/charly-<layer>.fish` |

Bash/zsh/sh on host targets use a per-candy fence pair so multiple
candies coexist in one rc file:

```
# opencharly:begin <layer> (managed by charly; do not edit inside this block)
<body>
# opencharly:end <layer>
```

`charly deploy del` strips just the candy's fence pair from the rc file
(without touching unrelated content). Fish always uses a per-candy
drop-in file (`conf.d/` is auto-sourced — no fence needed).

**Cross-environment behaviour** — declaring all four shells in one
candy covers the full cartesian product `{distros} × {host shells}`.
The runtime `command -v <shell>` probe at `target: local` / `target: vm`
deploy time skips snippets for shells that aren't installed on the
target — same precedent as how `aur:` skips on non-Arch.

**Field reference (intrinsic body OR per-shell sub-block):**

| Field | Type | Purpose |
|---|---|---|
| `init` | string (block scalar) | Snippet body. Required when the block is present. |
| `path_append` | `[]string` | PATH entries; rendered with shell-appropriate syntax. `~/`-prefix expands at install time. |
| `path` | string | Override destination file. `..`-traversal is rejected at validate time. |
| `priority` | int | Optional load order across candies contributing to the same shell. Default 50. |

**OCI label round-trip:** the merged shell config is baked into
`ai.opencharly.shell` at `charly box build` time and parsed back via
`ExtractMetadata` at deploy. `charly.yml` `shell:` overlays merge by
id (same replace/skip/append semantics as `eval:`).

**Migration:** `charly migrate` rewrites legacy `cmd:` shell-rc
heredoc tasks (matching the `# opencharly:begin direnv-hook` /
`# opencharly:begin ssh-auth-sock` fence patterns) into the structured
shell: schema. Idempotent.

---

## Cross-References

- `/charly-image:image` — Adding candies to box definitions; box composition; `data_image:` for data-only bundles; the full MCP-first authoring table including `box set`, `box add-candy`, `box rm-candy`, `box write`, `box cat`.
- `/charly-build:charly-mcp-cmd` — "Authoring tools" table exposing `candy.set`, `candy.add-rpm`, `candy.add-deb`, `candy.add-pac`, `candy.add-aur` as MCP tools; end-to-end build-from-scratch worked example.
- `/charly-build:generate` — What `charly box generate` actually emits; the per-verb emitter pipeline; `.build/<image>/` layout.
- `/charly-build:validate` — Validation rules (including per-verb task requirements).
- `/charly-build:new` — Scaffolding a new candy directory.
- `/charly-build:build` — Building images (`--no-cache` caveat; multi-stage scratch).
- `/charly-core:charly-config` — Cross-container `env_provide` / `mcp_provide` injection; `env_require` enforcement; `--update-all`; resource caps.
- `/charly-core:deploy` — `charly.yml` `provides:` section; tunnel is charly.yml-only.
- `/charly-eval:eval` — `eval:` field for declarative candy checks (file/port/http/...); embedded in the `ai.opencharly.eval` OCI label under the `layer` section. Candy eval checks default to `scope: build`; opt into `scope: deploy` to reference runtime vars like `${HOST_PORT:N}`. **Cross-distro package tests:** use `package_map:` on a `package:` check to resolve distro-specific package names (Fedora `openssh-server` vs Arch `openssh`); see the skill's "Cross-distro package names" section and the worked example in `candy/sshd/charly.yml`.
- `/charly-automation:sidecar` — Sidecars as `env_provide` participants (tailscale `TS_*` filtering).
- `/charly-build:secrets` — Credential store chain for `secret_accept` / `secret_require`.
- `/charly-selkies:chrome` — Canonical consumer of `env_accept` (proxy vars), cgroup resource caps, and a heavy user-phase copy/mkdir task list.
- `/charly-infrastructure:supervisord` — Event listener pattern triggered by resource caps.
- `/charly-tools:charly` — The charly-binary candy (composed by every charly-driving box). Paired with `/charly-coder:charly-mcp` which turns any box into an MCP server exposing the full charly CLI.
- `/charly-coder:charly-mcp` — Reference implementation of a meta-layer composition (`candy: [charly, supervisord]` — no install of its own, just wiring) with bind-mounted project directory and `CHARLY_PROJECT_DIR` env-var plumbing.
- `/charly-jupyter:notebook-templates` — Data-candy example.
- `/charly-internals:generate-source` — Internal architecture of the task emission pipeline (Go side).

---

## Cross-kind name reuse

A candy's name lives in its own namespace — same as `image:`, `pod:`, `vm:`, `k8s:`, `local:`, and `deploy:`. The same identifier (e.g. `charly-cachyos`) MAY exist as a candy at `candy/charly-cachyos/` AND a box entry `box.charly-cachyos` AND a deploy row `deploy.charly-cachyos` simultaneously. Verbs disambiguate by context. When `charly deploy add <name>` resolves a ref where both a box AND a candy with that name exist, box wins (box-first precedence); use `--add-candy <name>` to explicitly select the candy for an overlay. See CLAUDE.md "Cross-kind name reuse is permitted and encouraged" and `/charly-core:deploy`.

---

## When to Use This Skill

**MUST be invoked** for any task involving candy authoring, `charly.yml`, `task:`, `var:`, `pixi.toml`, `package.json`, `Cargo.toml`, or any file under `candy/`. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Author candies before adding them to boxes. See `/charly-image:image` (composition), `/charly-build:build` (building), `/charly-build:generate` (emission internals).

## Related skills

- `/charly-build:migrate` — `charly migrate` converts legacy flat-form candy definitions + raw-INI `service:` blocks into the canonical schema
- `/charly-internals:capabilities` — how the `service:` list is baked into the `LabelService` OCI label
- `/charly-internals:install-plan` — internal IR the loader feeds into build/deploy pipelines
