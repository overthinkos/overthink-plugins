---
name: image
description: |
  MUST be invoked before any work involving: the `ov image` command family, image definitions in image.yml, image inheritance, defaults, platforms, builder configuration, the image dependency graph, or the build/deploy scope boundary.
---

# ov image -- Family Overview + Image Composition

## Overview

`ov image` is the **only** command family that reads `image.yml`. It groups
every build-mode operation (build, generate, validate, list, merge, new,
inspect, pull) under a single namespace. All other `ov` commands read
exclusively from OCI labels embedded into built images + `deploy.yml` for
deployment overrides.

This scope boundary was introduced by the `ov image` refactor. Old top-level
invocations (`ov build`, `ov validate`, `ov list images`, `ov inspect`, etc.)
return Kong's `unexpected argument` error — there are no backward-compat shims.

An **image** is a named build target in `image.yml`. Images compose layers
into container images with configurable defaults, inheritance chains,
platform targets, and builder configurations. The `ov` CLI resolves
dependencies, generates Containerfiles, and builds images in the correct
order.

## The `ov image` Command Family

| Subcommand | Purpose | Skill |
|---|---|---|
| `ov image build` | Build container images from image.yml | `/ov:build` |
| `ov image generate` | Write `.build/` Containerfiles | `/ov:generate` |
| `ov image inspect` | Print resolved image config as JSON | `/ov:inspect` |
| `ov image list {images,layers,targets,services,routes,volumes,aliases}` | List components from image.yml | `/ov:list` |
| `ov image merge` | Merge small layers in a built image | `/ov:merge` |
| `ov image new layer <name>` | Scaffold a new layer directory | `/ov:new` |
| `ov image pull` | Fetch an image into local storage | `/ov:pull` |
| `ov image test` | Run declarative tests against a disposable container from a built image (reads the `org.overthinkos.tests` OCI label) | `/ov:test` |
| `ov image validate` | Check image.yml + layers | `/ov:validate` |

## Scope Boundary (Build vs. Deploy)

| | Reads `image.yml` | Reads OCI labels | Reads `deploy.yml` |
|---|---|---|---|
| `ov image …` | **Yes** (required) | Rarely | No |
| Everything else | **No** | Yes (required for deploy-mode) | Yes (overlay) |

If a new command needs to resolve layer dependencies, image inheritance, or
registry tag configuration, it must live under `ov image`. Any command that
operates on a running container or deployed image must go through
`ExtractMetadata` (labels) + deploy.yml — never `LoadConfig`.

When a deploy-mode command is run against an image that isn't in local
storage, `ExtractMetadata`/`EnsureImage` return `ErrImageNotLocal` and the
top-level error handler renders: *"image 'X' is not available locally. Run
'ov image pull X' to fetch it first."* See `/ov:pull` for the full sentinel
pattern.

## Project directory resolution

Every `ov image …` command resolves `image.yml` (and `build.yml`, `layers/`, etc.) **relative to the current working directory** — internally via `os.Getwd()` on every entry point. Five ways to override that default — three local, two remote:

```bash
# Local project — pick a directory on disk:
ov -C /path/to/overthink image list images          # short flag
ov --dir /path/to/overthink image list images       # long flag
OV_PROJECT_DIR=/path/to/overthink ov image list images   # env var

# Remote project — clone (or hit cache) and chdir into it:
ov --repo overthinkos/overthink image list images        # bare owner/repo → github.com/owner/repo@<default-branch>
ov --repo overthinkos/overthink@main image list images   # pinned ref
ov --repo default image list images                      # literal "default" → overthinkos/overthink
OV_PROJECT_REPO=overthinkos/overthink ov image list images
```

`--repo` and `--dir` are mutually exclusive (passing both exits with `ov: --repo and --dir are mutually exclusive`). All five paths are declared on the top-level `CLI` struct in `ov/main.go` and resolved by a single `os.Chdir(cli.Dir)` call **before** Kong dispatches the subcommand, so every existing `os.Getwd()` site picks up the new cwd — no per-command plumbing needed.

**Repo spec normalization** (in `ov/main_repo.go`):

- `default` → `github.com/overthinkos/overthink` at the default branch
- bare `owner/repo` → `github.com/owner/repo` (auto-prefix when first segment has no dot)
- bare `owner/repo@ref` → pinned to `ref`
- `host.tld/owner/repo[@ref]` → used literally (the dot in the host disambiguates)

Remote repos are cloned into `~/.cache/ov/repos/<repoPath>@<version>/` (override via `OV_REPO_CACHE`). The cache is shared with the existing remote-layer fetcher (`ov/refs.go`, `ov/refs_git.go`) — both go through `EnsureRepoDownloaded`.

**Canonical use case**: running `ov mcp serve` inside a container. The container's cwd is `/root` (or similar) where no `image.yml` exists. There are now three deployment patterns, in order of progressively less local setup:

1. **Bind-mount** — the original `ov-mcp` layer pattern. Host project mounted at `/project`, `OV_PROJECT_DIR=/project` set in the container env. Use this when you want the agent to read your in-flight local edits.

   ```bash
   ov config arch-ov --bind project=/home/you/overthink
   ov start arch-ov
   ov test mcp call arch-ov image.list.images '{}' --name ov
   ```

2. **Remote pin** — set `OV_PROJECT_REPO=overthinkos/overthink@<sha-or-ref>` in the container env. The agent reads from a pinned upstream version. No bind mount required.

3. **Auto-default** — `ov mcp serve` with no `OV_PROJECT_DIR`/`OV_PROJECT_REPO` and no local `image.yml` silently falls back to `github.com/overthinkos/overthink`. Pass `--no-default-repo` on the serve command to opt out (it then hard-fails with a clear error). This is the only command that auto-fetches; the top-level CLI stays opt-in.

The error messages are explicit when misconfigured: `cannot chdir to --dir "/missing": no such file or directory` or `reading image.yml: open /project/image.yml: no such file or directory` (when the bind mount exists but hasn't been populated). See `/ov:mcp` "Deployment: the `ov-mcp` layer" for the full bind-mount pattern and `/ov-dev:go` "main.go" for the implementation note (guarded by `TestOvDir_FlagChdir` + `TestOvDir_Errors` in `main_dir_test.go`, and `TestNormalizeRepoSpec` + `TestOvRepo_*` in `main_repo_test.go`).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| List images | `ov image list images` | Images from image.yml |
| List build targets | `ov image list targets` | Build targets in dependency order (includes auto-intermediates) |
| Inspect image | `ov image inspect <image>` | Print resolved config as JSON |
| Inspect field | `ov image inspect <image> --format <field>` | Print single field (tag, base, layers, ports, etc.) |
| Validate | `ov image validate` | Check image.yml + layers |
| Pull into local storage | `ov image pull <image>` | Fetch from registry so deploy-mode commands work |
| Run build-time tests | `ov image test <image>` | Runs the baked layer + image test sections in a disposable `podman run --rm` container; add `--include-deploy` to also run the deploy section. See `/ov:test`. |
| Pre-prime remote repo cache | `ov image fetch [<spec>]` | Clones (or hits cache) for the spec — defaults to `default` (overthinkos/overthink). Prints the cache path. |
| Force re-clone | `ov image refresh [<spec>]` | Removes the cache entry and re-clones. |

### Authoring (the MCP-first surface)

Each verb below is also auto-exposed as an MCP tool (`image.new.project`, `image.new.image`, `image.set`, `image.add-layer`, `image.rm-layer`, `image.write`, `image.cat`, `layer.set`, `layer.add-rpm`, …) via `ov/mcp_server.go`'s Kong reflection. So an LLM agent driving `ov mcp serve` can author a project from scratch over RPC.

| Action | Command |
|--------|---------|
| Scaffold a fresh project | `ov image new project <dir>` |
| Add an image entry | `ov image new image <name> --base <ref> --layers <a,b,c>` |
| Add a layer dir (stub `layer.yml`) | `ov image new layer <name>` |
| Edit a value in `image.yml` | `ov image set <dotpath> <yaml-value>` |
| Append a layer to an image | `ov image add-layer <image> <layer>` |
| Remove a layer from an image | `ov image rm-layer <image> <layer>` |
| Edit a value in `layers/<name>/layer.yml` | `ov layer set <name> <dotpath> <yaml-value>` |
| Append rpm/deb/pac/aur packages to a layer | `ov layer add-rpm <name> <pkg…>` (and `add-deb`, `add-pac`, `add-aur`) |
| Write any file under the project root | `ov image write <rel-path> [--content X \| --from-stdin]` |
| Read any file under the project root | `ov image cat <rel-path>` |

**Safety boundary**: `ov image write` / `ov image cat` resolve the path against `os.Getwd()` (the project root) and reject absolute paths or `..` traversal that would escape the root. They are the deliberate escape hatch for free-form auxiliary files (`pixi.toml`, `package.json`, `root.yml`, `*.service`, scripts) that the schema-aware setters don't cover.

**Comment preservation**: every YAML edit (`set`, `add-layer`, `rm-layer`, `add-rpm`, etc.) goes through the `yaml.v3` *node* API rather than the value API, so human-authored comments and key order are preserved across edits. Tested in `ov/yaml_setter_test.go` and `ov/scaffold_project_test.go`.

**Project scaffold contents**: `ov image new project` writes a minimal `image.yml` whose `defaults.format_config` references the upstream `build.yml` remotely (`@github.com/overthinkos/overthink/build.yml`), so new projects don't have to copy the canonical 1k-line build.yml. Replace with a local `build.yml` + drop the `format_config` field if you need custom distro/builder/init definitions.

## image.yml Structure

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
| `version` | `""` | CalVer version (`YYYY.DDD.HHMM`) of this image definition. Set manually |
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
| `user` | `"user"` | Username for non-root operations |
| `uid` | `1000` | User ID |
| `gid` | `1000` | Group ID |
| `merge` | `null` | Layer merge settings |
| `aliases` | `[]` | Command aliases |
| `builder` | `{}` | Build type → builder image map (inherited from base image + defaults). Keys match the `build.yml` `builder:` section — e.g., `builder.pixi` selects which image to use as the pixi builder |
| `builds` | `[]` | What this builder image can build: `pixi`, `npm`, `cargo`, `aur` (not inherited) |
| `env` | `[]` | Runtime env vars (`KEY=VALUE`). Not inherited from defaults |
| `env_file` | `""` | Path to `.env` file for runtime injection. Not inherited |
| `security` | `null` | Container security options. Overrides layer-level security |
| `network` | `string` | Container network mode (default: shared `ov` network; set `host` for host networking) |
| `vm` | `VmConfig` | VM settings: `disk_size`, `root_size`, `ram`, `cpus`, `rootfs`, `kernel_args`, `ssh_port`, `transport`, `firmware`, `network` |
| `libvirt` | `[]string` | Raw libvirt XML snippets for VM domain configuration |

## Builder and Builds

Builder images provide build tools (pixi, npm, cargo, yay) for multi-stage builds without bloating final images. Three fields control this:

- **`builder:`** on images — map of build type → builder image name. Inherited: image → base image → defaults → `{}`. The keys (`pixi`, `npm`, `cargo`, `aur`) match entries in `build.yml`'s `builder:` section — intentionally the same word, because both maps key on the same slot.
- **`builds:`** on builder images — list declaring what the builder can build. Not inherited.
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

  archlinux:
    base: "docker.io/library/archlinux:latest"
    distro: [archlinux]
    build: [pac]
    builder:
      pixi: archlinux-builder
      npm: archlinux-builder
      cargo: archlinux-builder
      aur: archlinux-builder

  archlinux-builder:
    base: archlinux              # inherits build: [pac] AND builder: from archlinux
    builds: [pixi, npm, cargo, aur]
    layers: [pixi, nodejs, build-toolchain, yay]

  arch-test:
    base: archlinux              # inherits builder: from archlinux
    build: [pac, aur]            # override to add aur format
    layers: [arch-pac-test, arch-aur-test]
```

Each build type resolves its builder independently: **`image.builder[type]` → `base_image.builder[type]` → `defaults.builder[type]` → `""`**. This means you can use `fedora-builder` for pixi but `archlinux-builder` for npm on the same image.

Self-reference protection: after merging defaults/base, any `builder` entry pointing to the image itself is filtered out. Builder images can't use themselves as builders.

Validation checks that every builder referenced in `builder:` declares the matching capability in `builds:`.

Source: `ov/generate.go` (`builderRefForFormat`), `ov/graph.go` (`ResolveImageOrder`, `ImageNeedsBuilder`), `ov/validate.go` (`validateBuilders`).

## Internal Base Images

When `base` references another image in `image.yml`, the generator resolves it to the full registry/tag and creates a build dependency. The referenced image must be built first.

```yaml
images:
  fedora:
    base: "quay.io/fedora/fedora:43"

  my-app:
    base: fedora        # References fedora image above
    layers: [my-layer]
```

## External Bases Require Explicit `distro:`

When `base` is a URL string (not the name of another image in `image.yml`), the generator treats it as **external** and does not inherit distro tags or build formats. This is the canonical gotcha for bootc images, which typically use `quay.io/fedora/fedora-bootc:43`:

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

Symptom without `distro:`: `ov image inspect <image>` shows `"Distro": null`. The generator's install_template Phase-2 branch short-circuits on `img.DistroDef == nil`, so **no layer `rpm:` install RUN steps are emitted**. The image builds cleanly but is missing every package from every layer that uses declarative `rpm:` sections. Explicit `cmd: dnf install …` tasks still run; the bug affects only declarative `rpm:`/`deb:`/`pac:` sections.

Internal bases (`base: fedora`) inherit `distro:` and `build:` from the parent image automatically — you only need explicit tags on images whose `base:` is a URL. Canonical worked example: `/ov-images:selkies-desktop-bootc`. Sibling bootc templates (`/ov-images:openclaw-browser-bootc`, `/ov-images:bazzite-ai`, `/ov-images:aurora`) currently lack `distro:` — they're `enabled: false` so haven't tripped the bug yet, but will need the declaration before they can be enabled.

## Intermediate Images

When multiple images share the same base and common layer prefixes, `ov` auto-generates intermediate images at branch points to maximize cache reuse.

```
fedora (external)
  -> fedora-supervisord (auto: pixi + python + supervisord)
     -> fedora-test (adds: traefik, testapi)
     -> openclaw (adds: nodejs, openclaw)
```

Auto-intermediates are marked with `Auto: true` and appear in `ov image list targets`.

### Algorithm

`ComputeIntermediates()` runs during generation:
1. `GlobalLayerOrder()` computes a deterministic layer ordering across all images, prioritizing layers by popularity (how many images need them) for cache efficiency.
2. Images are grouped by their direct parent (base). For each sibling group with 2+ images, a **prefix trie** is built from their relative layer sequences.
3. The trie is walked to detect branch points (where sibling layer sequences diverge). At each branch, an auto-intermediate image is created.
4. Original images are rebased to the nearest intermediate, so shared layers are built once.

Source: `ov/intermediates.go` (`ComputeIntermediates`, `GlobalLayerOrder`, `walkTrieScoped`).

## Versioning

CalVer: `YYYY.DDD.HHMM` (year, day-of-year, UTC time). Computed once per `ov image generate`.

| `tag` value | Generated tag(s) |
|-------------|-----------------|
| `"auto"` | `YYYY.DDD.HHMM` + `latest` |
| `"nightly"` | `nightly` only |
| `"1.2.3"` | `1.2.3` only |

Override: `ov image generate --tag <value>`.

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

These are the lowest priority in the env resolution chain. CLI flags (`-e`, `--env-file`) and workspace `.env` take precedence. See `/ov:config` and `/ov:start` for the full priority chain at config-time and run-time respectively.

Source: `ov/envfile.go` (`ResolveEnvVars`).

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

Source: `ov/security.go` (`CollectSecurity`).

## VM Configuration

For bootc images, the `vm:` section configures VM disk builds and runtime:

```yaml
images:
  my-bootc-image:
    bootc: true
    base: "quay.io/fedora/fedora-bootc:43"
    layers: [sshd, qemu-guest-agent, bootc-config]
    vm:
      disk_size: "10 GiB"
      ram: "4G"
      cpus: 2
      rootfs: ext4
      ssh_port: 2222
    libvirt:
      - "<filesystem type='mount'>...</filesystem>"
```

Resolution chain: **image-level vm: -> defaults vm: -> ov settings -> hardcoded defaults**. See `/ov:deploy` for full VM management documentation.

## OCI Labels

Every image `ov` builds carries a set of `org.overthinkos.*` OCI labels embedding the resolved image config so that `ov config` and `ov deploy` can work without the project source tree. The full list is assembled in `ov/labels.go`:

| Label | Contents |
|---|---|
| `org.overthinkos.volumes` | Volume declarations from the layer chain |
| `org.overthinkos.ports` | Ports + protocol annotations |
| `org.overthinkos.security` | `cap_add`, `devices`, `security_opt`, `mounts`, resource caps |
| `org.overthinkos.env` | Runtime env keys |
| `org.overthinkos.env_provides` | Cross-container env provides (resolved at deploy time) |
| `org.overthinkos.env_requires` | Declared env contracts (used for `ov config` hard-fail checks) |
| `org.overthinkos.env_accepts` | Opt-in allowlist for provides filtering |
| `org.overthinkos.mcp_provides` | Cross-container MCP server provides |
| `org.overthinkos.port_protos` | Port protocol annotations (non-default only) |

All of the above round-trip via `ov config`: the label is read from the image manifest and applied to deploy.yml + the quadlet. There is one deliberate exception.

### Tunnel is deploy.yml-only

`labels.go:238` **explicitly skips reading** any tunnel label when resolving an image's deploy config. Tunnels (Tailscale serve, Cloudflare tunnel) are treated as a **deployment** decision, not an image attribute — they live exclusively in `deploy.yml`. This was the deliberate design of commit `2759124` (tunnel→deploy.yml migration), motivated by three concerns:

1. **Per-instance divergence.** One selkies-desktop image may be deployed with a Tailscale tunnel in one environment and no tunnel in another. Baking the tunnel choice into the image forecloses that.
2. **`--update-all` safety.** Propagating config changes across deployed services must not accidentally rewrite tunnel settings from image labels and blow away per-instance overrides.
3. **Instance inheritance gap.** Tunnel config is **not** auto-inherited from the base `ov config <image>` call to an `ov config <image> -i <instance>` call. This is a deliberate gap — see `/ov-layers:selkies-desktop` (Multi-Instance Proxy Deployment) for the manual workaround and `/ov:deploy` (Instance Tunnel Inheritance) for the full lifecycle.

**Practical implication:** you can inspect an image's tunnel declaration with `ov image inspect <image>` and see nothing useful — that's correct. To see a tunnel's actual state, read `deploy.yml` directly (`ov deploy show <image>`) or the generated quadlet (`ov status <image>`).

## Common Workflows

### Add a New Image

Add an entry to `image.yml` with `base` and `layers`, then build:

```bash
# Edit image.yml
# Then:
ov image build my-new-image
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

- `/ov:build` -- `ov image build` (+ the `--no-cache` intermediate scratch-stage caveat)
- `/ov:generate` -- `ov image generate` (Containerfile generation including OCI label emission)
- `/ov:inspect` -- `ov image inspect` (resolved OCI label set)
- `/ov:list` -- `ov image list {images,layers,targets,services,routes,volumes,aliases}`
- `/ov:merge` -- `ov image merge` (post-build layer consolidation)
- `/ov:new` -- `ov image new layer <name>` (scaffold new layer directory)
- `/ov:pull` -- `ov image pull` (fetch into local storage; `ErrImageNotLocal` recovery)
- `/ov:validate` -- `ov image validate` (image.yml + layers consistency check)

### Related skills

- `/ov:layer` -- Layer definitions that compose into images (env_provides, env_requires, env_accepts, security resource caps)
- `/ov:deploy` -- Deploying built images (quadlet, bootc, tunnel lifecycle, instance tunnel inheritance)
- `/ov:config` -- `ov config` reads OCI labels + deploy.yml; tunnel is deploy.yml-only
- `/ov-dev:go` -- `LoadConfig`, `ExtractMetadata`, `EnsureImage`, `ErrImageNotLocal` source locations
- `/ov:test` — Image-level `tests:` (cross-layer invariants) and `deploy_tests:` (deploy-default checks shipped with the image). Both are embedded in the `org.overthinkos.tests` OCI label.
- `/ov:mcp` — if the image transitively bundles an mcp-providing layer (e.g. `jupyter`, `chrome-devtools-mcp`), the bundled layer's `mcp:` tests run as part of `ov test <image> --filter mcp`; see the skill for per-verb details and the port-publishing gotcha.
- `/ov-images:selkies-desktop-bootc` — canonical worked example for the external-base + explicit-`distro:` pattern.
- `/ov:vm` — `bootc: true` and `vm:` field consumers; covers the `vm.ssh_port` plumbing, `/dev:/dev` mount, and rootful storage refresh.

## When to Use This Skill

**MUST be invoked** when the task involves image definitions in image.yml, image inheritance, defaults, platforms, builder configuration, or the image dependency graph. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Define images before building. See also `/ov:layer` (layer authoring), `/ov:build` (building).
