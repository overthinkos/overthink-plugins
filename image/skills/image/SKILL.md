---
name: image
description: |
  MUST be invoked before any work involving: the `charly box` command family, image definitions in box.yml, image inheritance, defaults, platforms, builder configuration, the image dependency graph, or the build/deploy scope boundary.
---

# charly box -- Family Overview + Image Composition

## Overview

`charly box` is the **only** command family that reads `box.yml`. It groups
every build-mode operation (build, generate, validate, list, merge, new,
inspect, pull) under a single namespace. All other `charly` commands read
exclusively from OCI labels embedded into built images + `deploy.yml` for
deployment overrides.

Build-mode operations live only under `charly box`. Top-level invocations like
`charly build`, `charly validate`, `charly list images`, or `charly inspect` return Kong's
`unexpected argument` error.

An **image** is a named build target in `box.yml`. Images compose layers
into container images with configurable defaults, inheritance chains,
platform targets, and builder configurations. The `charly` CLI resolves
dependencies, generates Containerfiles, and builds images in the correct
order.

## The `charly box` Command Family

| Subcommand | Purpose | Skill |
|---|---|---|
| `charly box build` | Build container images from box.yml | `/charly-build:build` |
| `charly box generate` | Write `.build/` Containerfiles | `/charly-build:generate` |
| `charly box inspect` | Print resolved image config as JSON | `/charly-build:inspect` |
| `charly box list {images,layers,targets,services,routes,volumes,aliases}` | List components from box.yml | `/charly-build:list` |
| `charly box merge` | Merge small layers in a built image | `/charly-build:merge` |
| `charly box new candy <name>` | Scaffold a new layer directory | `/charly-build:new` |
| `charly box pull` | Fetch an image into local storage | `/charly-build:pull` |
| `charly eval box` | Run declarative tests against a disposable container from a built image (reads the `ai.opencharly.eval` OCI label) | `/charly-eval:eval` |
| `charly box validate` | Check box.yml + layers | `/charly-build:validate` |

## Scope Boundary (Build vs. Deploy)

| | Reads `box.yml` | Reads OCI labels | Reads `deploy.yml` |
|---|---|---|---|
| `charly box …` | **Yes** (required) | Rarely | No |
| Everything else | **No** | Yes (required for deploy-mode) | Yes (overlay) |

If a new command needs to resolve layer dependencies, image inheritance, or
registry tag configuration, it must live under `charly box`. Any command that
operates on a running container or deployed image must go through
`ExtractMetadata` (labels) + deploy.yml — never `LoadConfig`.

When a deploy-mode command is run against an image that isn't in local
storage, `ExtractMetadata`/`EnsureImage` return `ErrImageNotLocal` and the
top-level error handler renders: *"image 'X' is not available locally. Run
'charly box pull X' to fetch it first."* See `/charly-build:pull` for the full sentinel
pattern.

## Project directory resolution

Every `charly box …` command resolves `box.yml` (and `build.yml`, `candy/`, etc.) **relative to the current working directory** — internally via `os.Getwd()` on every entry point. Five ways to override that default — three local, two remote:

```bash
# Local project — pick a directory on disk:
charly -C /path/to/opencharly image list images          # short flag
charly --dir /path/to/opencharly image list images       # long flag
CHARLY_PROJECT_DIR=/path/to/opencharly charly box list boxes   # env var

# Remote project — clone (or hit cache) and chdir into it:
charly --repo overthinkos/overthink image list images        # bare owner/repo → github.com/owner/repo@<default-branch>
charly --repo overthinkos/overthink@main image list images   # pinned ref
charly --repo default image list images                      # literal "default" → overthinkos/overthink
CHARLY_PROJECT_REPO=overthinkos/overthink charly box list boxes
```

`--repo` and `--dir` are mutually exclusive (passing both exits with `charly: --repo and --dir are mutually exclusive`). All five paths are declared on the top-level `CLI` struct in `charly/main.go` and resolved by a single `os.Chdir(cli.Dir)` call **before** Kong dispatches the subcommand, so every existing `os.Getwd()` site picks up the new cwd — no per-command plumbing needed.

**Repo spec normalization** (in `charly/main_repo.go`):

- `default` → `github.com/overthinkos/overthink` at the default branch
- bare `owner/repo` → `github.com/owner/repo` (auto-prefix when first segment has no dot)
- bare `owner/repo@ref` → pinned to `ref`
- `host.tld/owner/repo[@ref]` → used literally (the dot in the host disambiguates)

Remote repos are cloned into `~/.cache/charly/repos/<repoPath>@<version>/` (override via `CHARLY_REPO_CACHE`). The cache is shared with the existing remote-layer fetcher (`charly/refs.go`, `charly/refs_git.go`) — both go through `EnsureRepoDownloaded`.

**Canonical use case**: running `charly mcp serve` inside a container. The container's cwd is `/workspace` (set by the `charly-mcp` layer's env + volume declaration). There are three deployment patterns, in order of progressively less local setup:

1. **Bind-mount** — the canonical `charly-mcp` pattern. Host project bind-mounted to the container's `/workspace`; volume NAME stays `project` for a stable deployer API. Use this when you want the agent to read your in-flight local edits.

   ```bash
   charly config charly-arch --bind project=/home/you/opencharly
   charly start charly-arch
   charly eval mcp call charly-arch box.list.boxes '{}' --name charly
   ```

2. **Remote pin** — set `CHARLY_PROJECT_REPO=overthinkos/overthink@<sha-or-ref>` in the container env. The agent reads from a pinned upstream version. No bind mount required.

3. **Auto-default** — `charly mcp serve` with no `box.yml` reachable at cwd silently falls back to `github.com/overthinkos/overthink`. The fallback fires whenever cwd lacks `box.yml`, regardless of whether `CHARLY_PROJECT_DIR` is set (the `charly-mcp` layer permanently sets `CHARLY_PROJECT_DIR=/workspace`, so a fallback gated on the env var being empty would never fire). Pass `--no-default-repo` on the serve command to opt out. Only `charly mcp serve` auto-fetches; the top-level CLI stays opt-in.

The error messages are explicit when misconfigured: `cannot chdir to --dir "/missing": no such file or directory`. See `/charly-build:charly-mcp-cmd` "Deployment: the `charly-mcp` layer" for the full bind-mount pattern and `/charly-internals:go` "main.go" for the implementation note (guarded by `TestCharlyDir_FlagChdir` + `TestCharlyDir_Errors` in `main_dir_test.go`, and `TestNormalizeRepoSpec` + `TestCharlyRepo_*` in `main_repo_test.go`).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `charly box list boxes` | Images from box.yml |
| List build targets | `charly box list targets` | Build targets in dependency order (includes auto-intermediates) |
| Inspect image | `charly box inspect <image>` | Print resolved config as JSON |
| Inspect field | `charly box inspect <image> --format <field>` | Print single field (tag, base, layers, ports, etc.) |
| Validate | `charly box validate` | Check box.yml + layers |
| Pull into local storage | `charly box pull <image>` | Fetch from registry so deploy-mode commands work |
| Run build-time tests | `charly eval box <image>` | Runs the baked layer + image test sections in a disposable `podman run --rm` container (build-scope only). For full-stack live eval against a running deployment, use `charly eval live <name>`. See `/charly-eval:eval`. |
| Pre-prime remote repo cache | `charly box fetch [<spec>]` | Clones (or hits cache) for the spec — defaults to `default` (overthinkos/overthink). Prints the cache path. |
| Force re-clone | `charly box refresh [<spec>]` | Removes the cache entry and re-clones. |

### Authoring (the MCP-first surface)

Each verb below is also auto-exposed as an MCP tool (`box.new.project`, `box.new.box`, `box.set`, `box.add-candy`, `box.rm-candy`, `box.write`, `box.cat`, `candy.set`, `candy.add-rpm`, …) via `charly/mcp_server.go`'s Kong reflection. So an LLM agent driving `charly mcp serve` can author a project from scratch over RPC.

| Action | Command |
|--------|---------|
| Scaffold a fresh project | `charly box new project <dir>` |
| Add an image entry | `charly box new box <name> --base <ref> --layers <a,b,c>` |
| Add a layer dir (stub `candy.yml`) | `charly box new candy <name>` |
| Edit a value in `box.yml` | `charly box set <dotpath> <yaml-value>` |
| Append a layer to an image | `charly box add-candy <image> <layer>` |
| Remove a layer from an image | `charly box rm-candy <image> <layer>` |
| Edit a value in `candy/<name>/candy.yml` | `charly candy set <name> <dotpath> <yaml-value>` |
| Append rpm/deb/pac/aur packages to a layer | `charly candy add-rpm <name> <pkg…>` (and `add-deb`, `add-pac`, `add-aur`) |
| Write any file under the project root | `charly box write <rel-path> [--content X \| --from-stdin]` |
| Read any file under the project root | `charly box cat <rel-path>` |

**Safety boundary**: `charly box write` / `charly box cat` resolve the path against `os.Getwd()` (the project root) and reject absolute paths or `..` traversal that would escape the root. They are the deliberate escape hatch for free-form auxiliary files (`pixi.toml`, `package.json`, `root.yml`, `*.service`, scripts) that the schema-aware setters don't cover.

**Comment preservation**: every YAML edit (`set`, `add-layer`, `rm-layer`, `add-rpm`, etc.) goes through the `yaml.v3` *node* API rather than the value API, so human-authored comments and key order are preserved across edits. Tested in `charly/yaml_setter_test.go` and `charly/scaffold_project_test.go`.

**Project scaffold contents**: `charly box new project` writes a minimal `charly.yml` with `discover: [box, candy]` + empty `box/`/`candy/` dirs. The default distro/builder/init/resource build vocabulary is EMBEDDED in the `charly` binary (`charly/build.yml`, `//go:embed`), so a new project is immediately usable with no `build.yml` to copy; declare `distro:`/`builder:`/`init:`/`resource:` (inline in `charly.yml` or an imported vocab file) only to extend or override the embedded default.

## box.yml Structure

```yaml
defaults:
  registry: ghcr.io/overthinkos
  tag: auto                    # CalVer: YYYY.DDD.HHMM
  platforms:
    - linux/amd64
    - linux/arm64
  build: [rpm]
  builder:                     # build type → builder image
    pixi: fedora-builder
    npm: fedora-builder
    cargo: fedora-builder
  merge:
    auto: false
    max_mb: 128

images:
  fedora:
    base: "quay.io/fedora/fedora:43"
    distro: ["fedora:43", fedora]

  fedora-builder:
    base: fedora
    builds: [pixi, npm, cargo]  # declares what this builder can build
    layers:
      - pixi
      - nodejs
      - build-toolchain

  my-app:
    base: fedora
    layers:
      - supervisord
      - traefik
      - my-service
    ports:
      - "8080:8080"
    env:
      - MY_VAR=value
    env_file: "~/.config/my-app/.env"
    security:
      cap_add: [SYS_PTRACE]
```

## Inheritance Chain

Every setting resolves through: **image -> defaults -> hardcoded fallback** (first non-null wins).

| Field | Default | Description |
|-------|---------|-------------|
| `enabled` | `true` | Set `false` to disable (skipped by generate, validate, list) |
| `version` | `""` | OPTIONAL dedicated CalVer (`YYYY.DDD.HHMM`). When set it IS the image's `ai.opencharly.version` label; when unset the label is derived as the highest layer version across the chain (`EffectiveVersion`, `charly/effective_version.go`). Layered images leave it unset (they derive — keeps the label content-stable); a layerless bare base on an EXTERNAL registry base needs it (else the label can't be derived) — `charly migrate` backfills those |
| `status` | `""` (= `testing`) | `working`, `testing`, or `broken`. Effective status = worst of image + all layers |
| `info` | `""` | Free-form description. Aggregated with layer-level info in OCI labels |
| `base` | `quay.io/fedora/fedora:43` | External OCI image or name of another image |
| `bootc` | `false` | Adds `bootc container lint`, enables disk image builds |
| `platforms` | `["linux/amd64", "linux/arm64"]` | Target architectures |
| `tag` | `"auto"` | Image tag. `"auto"` for CalVer |
| `registry` | `""` | Container registry prefix |
| `distro` | `[]` | Distro identity tags in priority order: `["fedora:43", fedora]`. For packages: first matching section wins (override). For tasks: additive. Inherited from base image |
| `build` | `["rpm"]` | Package formats tied to builder definitions: `[rpm]` or `[pac, aur]`. ALL formats installed in order. Valid: rpm, deb, pac, aur. Inherited from base image |
| `layers` | (required) | Layer list (image-specific, not inherited) |
| `ports` | `[]` | Runtime port mappings (`"host:container"` or `"port"`) |
| `user` | `"user"` | Username for non-root operations. See `user_policy:` — may be overridden at resolve time when adopt mode fires |
| `uid` | `1000` | User ID (may be overridden by `base_user:` under adopt) |
| `gid` | `1000` | Group ID (may be overridden by `base_user:` under adopt) |
| `user_policy` | `"auto"` | How to reconcile `user:` against the base image's pre-existing uid-1000 account. Values: `auto` / `adopt` / `create`. See "user_policy" section below |
| `merge` | `null` | Layer merge settings |
| `aliases` | `[]` | Command aliases |
| `builder` | `{}` | Build type → builder image map (inherited from base image + defaults). Keys match the `build.yml` `builder:` section — e.g., `builder.pixi` selects which image to use as the pixi builder |
| `builds` | `[]` | What this builder image can build: `pixi`, `npm`, `cargo`, `aur` (not inherited) |
| `env` | `[]` | Runtime env vars (`KEY=VALUE`). Not inherited from defaults |
| `env_file` | `""` | Path to `.env` file for runtime injection. Not inherited |
| `security` | `null` | Container security options. Overrides layer-level security |
| `network` | `string` | Container network mode (default: shared `charly` network; set `host` for host networking) |

VM-related fields (`vm`, `libvirt`) are not valid on kind:image entries — the loader rejects them. VMs are declared as `kind: vm` entities in `vm.yml` — see `/charly-vm:vms-catalog` for authoring and `/charly-build:migrate` for `charly migrate` conversion of legacy configs. `bootc: true` stays on kind:image entries (marks the image as a bootable container); a separate `kind: vm` entity with `source.kind: bootc` references it.

## Builder and Builds

Builder images provide build tools (pixi, npm, cargo, yay) for multi-stage builds without bloating final images. Three fields control this:

- **`builder:`** on images — map of build type → builder image name. Inherited: image → base image → defaults → `{}`. The keys (`pixi`, `npm`, `cargo`, `aur`) match entries in `build.yml`'s `builder:` section — intentionally the same word, because both maps key on the same slot.
- **`produce:`** on builder images — list declaring what the builder can build. Not inherited.
- **`build:`** — package formats tied to builder definitions (`rpm`, `deb`, `pac`, `aur`). ALL formats installed in order. Inherited from base image. Default: `[rpm]`.
- **`distro:`** — distro identity tags in priority order (`["fedora:43", fedora]`). First matching section overrides packages. Inherited from base image.

```yaml
defaults:
  builder:
    pixi: fedora-builder
    npm: fedora-builder
    cargo: fedora-builder

images:
  fedora-builder:
    base: fedora
    builds: [pixi, npm, cargo]
    layers: [pixi, nodejs, build-toolchain]

  arch:
    base: "quay.io/archlinux/archlinux:base-20260525.0.535911"
    distro: [arch]
    build: [pac]
    builder:
      pixi: arch-builder
      npm: arch-builder
      cargo: arch-builder
      aur: arch-builder

  arch-builder:
    base: arch              # inherits build: [pac] AND builder: from arch
    builds: [pixi, npm, cargo, aur]
    layers: [pixi, nodejs, build-toolchain, yay]

  arch-test:
    base: arch              # inherits builder: from arch
    build: [pac, aur]            # override to add aur format
    layers: [arch-pac-test, arch-aur-test]
```

Each build type resolves its builder independently: **`image.builder[type]` → `base_image.builder[type]` → `defaults.builder[type]` → `""`**. This means you can use `fedora-builder` for pixi but `arch-builder` for npm on the same image.

Self-reference protection: after merging defaults/base, any `builder` entry pointing to the image itself is filtered out. Builder images can't use themselves as builders.

Validation checks that every builder referenced in `builder:` declares the matching capability in `produce:`.

Source: `charly/generate.go` (`builderRefForFormat`), `charly/graph.go` (`ResolveImageOrder`, `ImageNeedsBuilder`), `charly/validate.go` (`validateBuilders`).

## Internal Base Images

When `base` references another image in `box.yml`, the generator resolves it to the full registry/tag and creates a build dependency. The referenced image must be built first.

```yaml
images:
  fedora:
    base: "quay.io/fedora/fedora:43"

  my-app:
    base: fedora        # References fedora image above
    layers: [my-layer]
```

## user_policy: adopt vs create

`user_policy:` cleanly handles base images that ship a pre-existing uid-1000 account (notably Ubuntu 24.04's `ubuntu:ubuntu`). A plain `getent passwd $UID || useradd …` bootstrap short-circuits on such accounts, leaving the image's configured `user:` never created — sudoers, `${HOME}`, npm prefix, etc. would then break because they assume the configured name exists.

The mechanism: a **declarative** fact (what the base image ships, in `build.yml distro.<name>.base_user:` — see `/charly-build:build`) + an **image-level policy** (how to reconcile with the image's `user:` field).

### Policy values

| Policy | Behavior | Failure mode |
|--------|----------|--------------|
| `auto` (default) | If `base_user:` is declared for the image's distro AND the image didn't explicitly set `user:`, adopt the base_user. Otherwise create the configured user. | Never fails — falls through gracefully. |
| `adopt` | Always adopt. Error if the distro has no `base_user:` declaration. | Hard error at config resolve time. Use when you specifically need to lock adopt semantics. |
| `create` | Always create the configured user. | Build fails if `useradd` collides (should never happen on a "create" distro). |

### Decision matrix per base image

| Base image | `base_user` declared? | `user_policy: auto` outcome | Resolved user |
|---|---|---|---|
| `/charly-distros:fedora` | no | create | `user` |
| `/charly-distros:arch` | no | create | `user` |
| `/charly-distros:debian` | no | create | `user` |
| `/charly-distros:ubuntu` | **yes** (`ubuntu:1000:/home/ubuntu`) | adopt | `ubuntu` |

This is why `ubuntu-coder`'s resolved identity is `ubuntu:/home/ubuntu` while the other three coder images are `user:/home/user`. The box.yml for all four coder images is identical on the user-related fields (no explicit `user:`); the policy + base_user together decide the outcome.

### How resolution flows (`charly/config.go ResolveImage`)

1. Resolve `User`, `UID`, `GID` from defaults → image overrides → hardcoded fallback `user` / `1000` / `1000`.
2. Load the distro config (`DistroConfig` from `build.yml`), resolve the image's `DistroDef` by walking `distro:` tags.
3. Apply `user_policy`:
   - `adopt` → overwrite User/UID/GID/Home with the distro's `BaseUser`, set `ResolvedImage.UserAdopted = true`.
   - `auto` → same overwrite IF `base_user` exists AND the image didn't explicitly set `user:`.
   - `create` → no-op.
4. `writeBootstrap` (`charly/generate.go`) keys on `UserAdopted`: adopt emits only a comment; create emits an idempotent `useradd` (see `/charly-build:generate`).

### Live verification

```bash
charly box inspect ubuntu-coder | grep -E '"User"|"UID"|"Home"|UserAdopted'
# "User": "ubuntu",
# "UID": 1000,
# "Home": "/home/ubuntu",
# "UserAdopted": true,

charly box inspect debian-coder | grep -E '"User"|"UID"|"Home"|UserAdopted'
# "User": "user",
# "UID": 1000,
# "Home": "/home/user",
# "UserAdopted": false,
```

### Layer consequences

Adopt mode means `resolved.User` is not a stable string across distros. Layers that reference the uid-1000 account by name must NOT hardcode `user` — use `${USER}` where the generator substitutes (task fields like `user: ${USER}`), or use `getent passwd 1000 | cut -d: -f1` inside `cmd:` blocks (where the generator does NOT substitute — bash sees the script verbatim). The canonical getent example is `/charly-coder:sshd`'s sudoers task.

See also `/charly-distros:ubuntu` (canonical adopt consumer), `/charly-build:build` "base_user:", `/charly-internals:go` "ResolvedImage.UserAdopted".

## External Bases Require Explicit `distro:`

When `base` is a URL string (not the name of another image in `box.yml`), the generator treats it as **external** and does not inherit distro tags or build formats. This is the canonical gotcha for bootc images, which typically use `quay.io/fedora/fedora-bootc:43`:

```yaml
# ❌ BROKEN — Distro resolves to null, no RPM installs emitted
my-bootc-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  layers: [sshd, qemu-guest-agent, ffmpeg]

# ✓ CORRECT — explicit distro: tags matching the base
my-bootc-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  distro: ["fedora:43", fedora]
  layers: [sshd, qemu-guest-agent, ffmpeg]
```

Symptom without `distro:`: `charly box inspect <image>` shows `"Distro": null`. The generator's install_template Phase-2 branch short-circuits on `img.DistroDef == nil`, so **no layer `rpm:` install RUN steps are emitted**. The image builds cleanly but is missing every package from every layer that uses declarative `rpm:` sections. Explicit `cmd: dnf install …` tasks still run; the bug affects only declarative `rpm:`/`deb:`/`pac:` sections.

Internal bases (`base: fedora`) inherit `distro:` and `build:` from the parent image automatically — you only need explicit tags on images whose `base:` is a URL. The `quay.io/fedora/fedora-bootc:43` example above is the canonical pattern: an external bootc base must declare its own `distro:` tags, or the generator sees `Distro: null` and emits no rpm-install RUN steps.

## Intermediate Images

When multiple images share the same base and common layer prefixes, `charly` auto-generates intermediate images at branch points to maximize cache reuse.

```
fedora (external)
  -> fedora-supervisord (auto: pixi + python + supervisord)
     -> fedora-test (adds: traefik, testapi)
     -> openclaw (adds: nodejs, openclaw)
```

Auto-intermediates are marked with `Auto: true` and appear in `charly box list targets`.

### Algorithm

`ComputeIntermediates()` runs during generation:
1. `GlobalLayerOrder()` computes a deterministic layer ordering across all images, prioritizing layers by popularity (how many images need them) for cache efficiency.
2. Images are grouped by their direct parent (base). For each sibling group with 2+ images, a **prefix trie** is built from their relative layer sequences.
3. The trie is walked to detect branch points (where sibling layer sequences diverge). At each branch, an auto-intermediate image is created.
4. Original images are rebased to the nearest intermediate, so shared layers are built once.

Source: `charly/intermediates.go` (`ComputeIntermediates`, `GlobalLayerOrder`, `walkTrieScoped`).

## Versioning

CalVer: `YYYY.DDD.HHMM` (year, day-of-year, UTC time). Computed once per `charly box generate`.

| `tag` value | Generated tag(s) |
|-------------|-----------------|
| `"auto"` | `YYYY.DDD.HHMM` + `latest` |
| `"nightly"` | `nightly` only |
| `"1.2.3"` | `1.2.3` only |

Override: `charly box generate --tag <value>`.

## Runtime Environment Variables

The `env` and `env_file` fields inject environment variables into containers at runtime (not build time):

```yaml
images:
  my-app:
    env:
      - DB_HOST=localhost
      - LOG_LEVEL=info
    env_file: "~/.config/my-app/.env"
```

These are the lowest priority in the env resolution chain. CLI flags (`-e`, `--env-file`) and workspace `.env` take precedence. See `/charly-core:charly-config` and `/charly-core:start` for the full priority chain at config-time and run-time respectively.

Source: `charly/envfile.go` (`ResolveEnvVars`).

## Security Configuration

Image-level `security:` overrides layer-level security settings:

```yaml
images:
  my-app:
    security:
      privileged: true
      cap_add: [SYS_ADMIN]
      devices: [/dev/fuse]
      security_opt: [label:disable]
```

Image `security.privileged` replaces the layer-derived value. `cap_add`, `devices`, `security_opt` are appended to layer-collected values (deduplicated). Applied as container run arguments at runtime (not build time).

Source: `charly/security.go` (`CollectSecurity`).

## VM Configuration

VMs are **not** configured on kind:image entries. The `image.vm:` and `image.libvirt:` fields are rejected at load time. VM primitives are declared as `kind: vm` entities in `vm.yml`:

```yaml
# vm.yml
vms:
  my-bootc-vm:
    source:
      kind: bootc
      box: my-bootc-image        # references the kind:image entry above (must have bootc: true)
    disk_size: 10 GiB
    ram: 4G
    cpus: 2
    libvirt:
      devices:
        filesystems: [{type: mount, source: ..., target: ...}]
```

See `/charly-vm:vms-catalog` for the full VmSpec schema, `/charly-vm:vm` for the `charly vm build/create/ssh` command family, and `/charly-build:migrate` for `charly migrate` to convert legacy `image.vm:` / `image.libvirt:` fields to the new schema.

## OCI Labels

Every image `charly` builds carries a set of `ai.opencharly.*` OCI labels embedding the resolved image config so that `charly config` and `charly deploy` can work without the project source tree. The full list is assembled in `charly/labels.go`:

| Label | Contents |
|---|---|
| `ai.opencharly.volume` | Volume declarations from the layer chain |
| `ai.opencharly.port` | Ports + protocol annotations |
| `ai.opencharly.security` | `cap_add`, `devices`, `security_opt`, `mounts`, resource caps |
| `ai.opencharly.env` | Runtime env keys |
| `ai.opencharly.env_provide` | Cross-container env provides (resolved at deploy time) |
| `ai.opencharly.env_require` | Declared env contracts (used for `charly config` hard-fail checks) |
| `ai.opencharly.env_accept` | Opt-in allowlist for provides filtering |
| `ai.opencharly.mcp_provide` | Cross-container MCP server provides |
| `ai.opencharly.port_proto` | Port protocol annotations (non-default only) |
| `ai.opencharly.platform.distro` | Distro identity (e.g. `["arch"]`) — first match picks bootstrap/format templates |
| `ai.opencharly.platform.format` | Package formats installed (`pac`, `rpm`, `deb`, `pixi`, `aur`, …) |
| `ai.opencharly.builder.use` | Consumer-side routing map: format → builder-image name |
| `ai.opencharly.builder.provide` | Producer-side capability list: formats this image can build for others |

All of the above round-trip via `charly config`: the label is read from the image manifest and applied to deploy.yml + the quadlet. There is one deliberate exception.

### Tunnel is deploy.yml-only

`labels.go:238` **explicitly skips reading** any tunnel label when resolving an image's deploy config. Tunnels (Tailscale serve, Cloudflare tunnel) are treated as a **deployment** decision, not an image attribute — they live exclusively in `deploy.yml`. This was the deliberate design of commit `2759124` (tunnel→deploy.yml migration), motivated by three concerns:

1. **Per-instance divergence.** One selkies-desktop image may be deployed with a Tailscale tunnel in one environment and no tunnel in another. Baking the tunnel choice into the image forecloses that.
2. **`--update-all` safety.** Propagating config changes across deployed services must not accidentally rewrite tunnel settings from image labels and blow away per-instance overrides.
3. **Instance inheritance gap.** Tunnel config is **not** auto-inherited from the base `charly config <image>` call to an `charly config <image> -i <instance>` call. This is a deliberate gap — see `/charly-selkies:selkies-labwc` (Multi-Instance Proxy Deployment) for the manual workaround and `/charly-core:deploy` (Instance Tunnel Inheritance) for the full lifecycle.

**Practical implication:** you can inspect an image's tunnel declaration with `charly box inspect <image>` and see nothing useful — that's correct. To see a tunnel's actual state, read `deploy.yml` directly (`charly deploy show <image>`) or the generated quadlet (`charly status <image>`).

## Common Workflows

### Add a New Image

Add an entry to `box.yml` with `base` and `layers`, then build:

```bash
# Edit box.yml
# Then:
charly box build my-new-image
```

### Layer Images (inheritance)

Set `base` to another image name:

```yaml
images:
  nvidia:
    base: fedora
    layers: [cuda]
    platforms: [linux/amd64]

  ml-workstation:
    base: nvidia
    layers: [python-ml, jupyter]
```

### Disable an Image

```yaml
images:
  experimental:
    enabled: false
    base: fedora
    layers: [experimental-layer]
```

## Cross-References

### Family subcommand skills

- `/charly-build:build` -- `charly box build` (+ the `--no-cache` intermediate scratch-stage caveat)
- `/charly-build:generate` -- `charly box generate` (Containerfile generation including OCI label emission)
- `/charly-build:inspect` -- `charly box inspect` (resolved OCI label set)
- `/charly-build:list` -- `charly box list {images,layers,targets,services,routes,volumes,aliases}`
- `/charly-build:merge` -- `charly box merge` (post-build layer consolidation)
- `/charly-build:new` -- `charly box new candy <name>` (scaffold new layer directory)
- `/charly-build:pull` -- `charly box pull` (fetch into local storage; `ErrImageNotLocal` recovery)
- `/charly-build:validate` -- `charly box validate` (box.yml + layers consistency check)

### Related skills

- `/charly-image:layer` -- Layer definitions that compose into images (env_provide, env_require, env_accept, security resource caps)
- `/charly-core:deploy` -- Deploying built images (quadlet, bootc, tunnel lifecycle, instance tunnel inheritance)
- `/charly-core:charly-config` -- `charly config` reads OCI labels + deploy.yml; tunnel is deploy.yml-only
- `/charly-internals:go` -- `LoadConfig`, `ExtractMetadata`, `EnsureImage`, `ErrImageNotLocal` source locations
- `/charly-eval:eval` — Image-level `eval:` (cross-layer invariants) and `deploy_eval:` (deploy-default checks shipped with the image). Both are embedded in the `ai.opencharly.eval` OCI label.
- `/charly-build:charly-mcp-cmd` — if the image transitively bundles an mcp-providing layer (e.g. `jupyter`, `chrome-devtools-mcp`), the bundled layer's `mcp:` tests run as part of `charly eval live <image> --filter mcp`; see the skill for per-verb details and the port-publishing gotcha.
- `/charly-vm:vm` — `charly vm build/create/start/stop/ssh` command family; reads `vm.yml`, not `box.yml`. Covers BIOS vs UEFI firmware, virtio-gpu video model, bootc caveats (rootful storage refresh, `-v /dev:/dev` loopback).
- `/charly-vm:vms-catalog` — authoring reference for the `kind: vm` entity schema.
- `/charly-build:migrate` — `charly migrate` converts legacy VM fields to `vm.yml`.

## Cross-kind name reuse

The `image:` map's namespace is independent of `candy/`, `pod:`, `vm:`, `k8s:`, `local:`, and `deploy:`. The same name MAY exist across all of them. Authoring verbs (`charly box set`, `charly box new box`, `charly box add-candy`, `charly box rm-candy`, `charly box new project`) write exclusively to `charly.yml` — per-kind `box.yml` is reachable only via the `import:` statement from `charly.yml`, never as a default authoring target. Missing `charly.yml` → hard error pointing at `charly box new project .` or `charly migrate`. See CLAUDE.md "Cross-kind name reuse".

### Files are generic kind-containers (per-kind filenames are a convenience)

Every YAML file is a generic, kind-agnostic container — the loader routes each document by its top-level kind-key (its SHAPE), **NEVER by filename**. So ANY file may hold ANY mix of kinds. Splitting entities into per-kind sibling files named for their kind (`box.yml` for boxes, `vm.yml` for VMs, `deploy.yml` for deploys, …) is a pure user **CONVENIENCE** you express in `charly.yml`'s `import:` (and, for candy directories, `discover:`) — it is never required, and the code hardcodes no per-kind filename. **`charly.yml` is the only filename the code knows**; everything else (which files to `import:`, which directories + manifest names to `discover:`) is configured there. Inline maps in `charly.yml` and per-kind splits load identically. `discover:` is a flat generic scan-spec list (`- {path, recursive, manifest}`); the manifest defaults to `candy.yml` but is overridable per spec. Migration of legacy configs: `charly migrate` (idempotent). See `/charly-build:migrate`, `/charly-internals:go`.

## The `import:` statement (composition + namespaces)

`import:` is the **single** composition statement — a **list**, one statement per project. Each list item is one of two shapes:

| Shape | YAML | Semantics |
|---|---|---|
| **Flat** | a bare string — `- build.yml`, `- '@github.com/owner/repo/build.yml:vTAG'` | Merge the referenced file's entities into THIS project's root namespace (root-wins). Use for same-repo per-kind file splits AND the shared `build.yml` distro/builder/init vocabulary. |
| **Namespaced** | a single-key map — `- {cachyos: box/cachyos}`, `- {charly: ../..}`, `- {base: '@github.com/owner/repo:vTAG'}` | Mount another project as an isolated child namespace under `alias`; its entries are NOT flat-merged, they're referenced QUALIFIED as `alias.entry`. |

### Qualified refs (`ns.entry`)

A namespaced import is reached through a dotted ref everywhere a name is resolved — `base:`, `builder:` map values, deploy cross-refs:

```yaml
import:
  - build.yml                       # flat — shared vocabulary
  - box.yml                       # flat — this repo's own kind:image entries
  - cachyos: box/cachyos          # namespaced child import

box:
  versa:
    base: cachyos.cachyos           # the `cachyos` image inside the `cachyos` namespace
```

Resolution is **namespace-relative**, exactly like Go package-member access: a bare ref inside a namespace resolves within that namespace first; a qualified ref descends one level per dot (`a.b.c` → namespace `a`, then `b`, then leaf `c`). The `arch` and `fedora` submodules are SELF-CONTAINED (`import: []`): their base/builder stacks are bare-local, so their images write `base: arch` / `base: fedora` and route builders to a bare-local `arch-builder` / `fedora-builder`. The `cachyos` submodule imports the **`overthinkos/arch`** submodule under the `arch` namespace, so it reaches the Arch base/builder as `arch.arch` / `arch.arch-builder`. The main repo, in turn, imports all three submodules (`arch` / `cachyos` / `fedora` namespaces) to reference their relocated boxes.

### Inheritance across a namespace boundary

- **`distro:` / `build:`** are VALUES (distro tags, package formats) → inherited across a namespace boundary, so a `base: cachyos.cachyos` image still picks up cachyos's `distro:`/`build:`.
- **`builder:`** is a map of REFS relative to the BASE's namespace → it does **NOT** cross the boundary. A consumer image that builds a multi-stage format declares its OWN `builder:` map, qualified to the right builder (e.g. the `cachyos` base, which imports the `arch` namespace, declares `builder: {pixi: arch.arch-builder}`). This avoids leaking a base-namespace-relative ref into a consumer where that namespace doesn't exist.

A repo reached via two import paths (e.g. `arch`, reached both directly as main → `arch` and transitively as main → `cachyos` → `arch`) — or an import cycle between two projects that import each other — is resolved to a single materialization at load time **by repo identity, not pinned version** — see `/charly-internals:go` "import-namespace loader". The consequence for authors: **the importing project's namespace pins win**. When an imported namespace's release imports your repo back (`<root>: @…:<someOldPin>`), that back-reference resolves to YOUR local working tree (the root), NOT the old pinned snapshot — so a stale transitive pin in a published submodule release can never drag a divergent (or stale-schema) version of your own repo into the load.

**`repo:` (optional root-only field).** Declare your project's canonical repo identity at the top of `charly.yml` (`repo: github.com/overthinkos/overthink`) so the loader recognizes a transitive back-import of your repo and short-circuits it to the local tree. When omitted, the loader infers it from `git remote origin`; absent both, the cycle-break degrades to version-keyed behavior. The field is purely additive (no migration needed).

### Layer-version resolution across namespaces — per-entity version

A namespace is imported to provide bases/builders; the resolver fetches ONLY the layers reachable from the enabled images' `base:`/`builder:` chains (reachability-scoped collection) — a namespace's unreferenced images and its `kind:local` templates are not pulled. The git `:vTAG` on a layer ref is only the FETCH coordinate; the layer's OWN `version:` (read after fetch) is the identity. So when the SAME layer is referenced via two different repo git tags but its `version:` is unchanged (a re-tag for an unrelated push), the resolver picks one materialization with NO warning. Only when a layer resolves to two genuinely different per-entity versions (a family pinned to a newer layer than the shared infra it composes) does it **warn once** (naming both per-entity versions) and use the **newest** (highest CalVer). Run `charly box reconcile` to align the on-disk git-tag pins and clear any warning. See `/charly-internals:go` "Remote-layer resolver", `/charly-build:reconcile`.

## Base stacks live in their distro submodules

The arch and fedora base-distro stacks are no longer carried by the main repo — each is owned by its `box/<distro>` submodule:

- **`box/arch`** owns `arch` + `arch-builder` (+ `cuda-arch-builder`), bare-local, and is SELF-CONTAINED (`import: []`).
- **`box/fedora`** owns `fedora` + `fedora-builder` + `fedora-nonfree` (+ the `nvidia` / `python-ml` GPU bases), bare-local, and is SELF-CONTAINED (`import: []`).
- **`box/cachyos`** owns the `cachyos` base (+ the pacstrap pair and the selkies GPU desktops) and imports the `arch` namespace to reach `arch.arch` / `arch.arch-builder` / `arch.cuda-arch-builder`.

The main repo imports all three submodules (`arch` / `cachyos` / `fedora` namespaces) to reference their relocated boxes from its own `eval`/`vm`/`local`/`k8s`/`android` entities (one-directional — the submodules import nothing back from main). The distro/builder/init build vocabulary is embedded in the `charly` binary (no `build.yml` import). See `/charly-distros:arch`, `/charly-distros:fedora`, `/charly-distros:cachyos`.

## When to Use This Skill

**MUST be invoked** when the task involves image definitions in box.yml, image inheritance, defaults, platforms, builder configuration, or the image dependency graph. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Define images before building. See also `/charly-image:layer` (layer authoring), `/charly-build:build` (building).

## Related skills

- `/charly-build:migrate` — `charly migrate` converts legacy `box.yml` into `image:` entries in `charly.yml`
- `/charly-internals:capabilities` — OCI label contract emitted at build time and consumed by deploy commands

## Live-deploy verification is mandatory (see `/charly-eval:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/charly-internals:disposable`). Use `charly update <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `charly deploy add <name> <ref> --disposable` or mark a VM in vm.yml.

**After committing the source-level fix, `charly update` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
