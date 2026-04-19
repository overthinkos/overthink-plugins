---
name: go
description: |
  Go CLI development: building the ov binary, running tests, understanding
  the source code structure.
  MUST be invoked before reading or modifying any Go source file in ov/.
---

# Go - CLI Development

## Overview

The `ov` CLI is a Go program in the `ov/` directory. It uses the Kong CLI framework, go-containerregistry for OCI operations, and YAML parsing for configuration. All computation, validation, and building logic lives in Go. Taskfiles are used only for bootstrapping (building ov itself).

### YAML surface ↔ Go identifier convention

The codebase keeps wire format (YAML keys) and internal names (Go fields/types) in strict symmetry — plural YAML keys get plural Go identifiers, singular get singular. When the `builders:` top-level key was renamed to `builder:` in `build.yml` and `image.yml`, the rename propagated to `BuildersMap → BuilderMap`, `ImageConfig.Builders → Builder`, `BuilderConfig.Builders → Builder`, `DistroConfig.Distros → Distro`, `InitConfig.Inits → Init`, and the OCI label constant `LabelBuilders → LabelBuilder` (value `org.overthinkos.builder`). The rule: if you change a YAML tag, also rename the Go identifier. Tests enforce this indirectly — struct literals won't compile if they disagree.

### Kong `default:"withargs"` for parent+leaf commands

Kong normally treats a struct as either a branch (has child `cmd:""` subcommands) OR a leaf (accepts `arg:""` positionals and has a `Run()` method) — not both. When you want both shapes on the same parent command (e.g., `ov test <image>` runs tests AND `ov test cdp …` dispatches to a subcommand), tag the default child with `default:"withargs"`. Kong then dispatches to that child when the first token doesn't match a subcommand name, passing positional args/flags through.

Two uses in the codebase:
- `ov/config_image.go:14-21` — `ImageConfigCmd.Setup` is the default; `ov config <image>` routes through `ImageConfigSetupCmd` while `ov config mount|status|…` dispatch explicitly.
- `ov/test_cmd.go:22-31` — `TestCmd.Run` is the default; `ov test <image>` runs declarative tests while `ov test cdp|wl|dbus|vnc …` dispatch explicitly.

Tradeoff: a subcommand name shadows a positional value with the same text. `ov test cdp` always dispatches to the cdp subcommand — if an image is literally named `cdp`, use the explicit `ov test run cdp` form.

### Mode purity: `LoadConfig` must NOT read `deploy.yml`

OCI labels are written exclusively from `image.yml` + `layer.yml` at `ov image build` / `ov image generate` time. `deploy.yml` is deploy-mode state and must never bleed into the baked image. The key guarantee lives in `ov/config.go:LoadConfig` — it calls `LoadConfigRaw` only, with no `MergeDeployOverlay`.

**The rule**: every build-mode command (anything under `ov image …`) calls `LoadConfig`. If you ever re-introduce `MergeDeployOverlay` inside `LoadConfig`, you will silently contaminate OCI labels with whatever is in the user's local `deploy.yml` — exactly the bug that made images bake `ports: ["5900:5900","9250:9222"]` from a stale `deploy.yml` entry instead of the `image.yml`-declared `["5900:5900","9222:9222","9224:9224"]`.

Deploy-mode commands (`ov config`, `ov start`, `ov shell`, `ov cmd`, `ov service`, `ov vm create`, …) read labels via `ExtractMetadata` and then apply the deploy overlay explicitly via `MergeDeployOntoMetadata(meta, dc, instance)`. This split is load-bearing — never collapse it.

### Self-exec coordination: host → container AND host → host

The `ov` binary self-execs in two distinct directions.

**Host → container** — the host `ov` delegates to a container-baked `ov` via `exec … ov <subcommand>`. Three sites today:
- `ov/notify.go:20` — best-effort desktop notification via in-container `ov test dbus notify`.
- `ov/dbus.go:195` — strict in-container `ov test dbus notify` with gdbus fallback.
- `ov/dbus.go:229` — generic D-Bus call via in-container `ov test dbus call`.

**Host → host** — the test runner spawns the same host `ov` binary as a subprocess to execute cdp/wl/dbus/vnc declarative verbs:
- `ov/testrun_ov_verbs.go` — the `runOvVerb` dispatcher builds `ov test <verb> <method> <image> [args…]` argv and runs it via `exec.CommandContext`, feeding stdout/stderr through the existing matcher pipeline. `findOvBinary()` prefers `os.Executable()` so tests invoke the same build that collected them, falling back to `$PATH`.

**The rule:** whenever you rename a subcommand path crossed by any of these self-exec sites, edit the host-side invocation strings AND plan a coordinated rebuild of every image that bakes the `ov` layer (affected images: grep `image.yml` for `- ov$`). For host→host sites the rebuild doesn't matter — it's the same binary — but the method-name allowlists in `testrun_ov_verbs.go` must stay in lockstep with the actual `ov test cdp|wl|dbus|vnc` subcommand tree.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build | `task build:ov` | Compile to `bin/ov` and install as Arch package |
| Install | `task build:install` | Install ov as Arch package (uses pre-built binary) |
| Run tests | `cd ov && go test ./...` | Run all tests |
| Run specific test | `cd ov && go test -run TestName ./...` | Run single test |
| Vet | `cd ov && go vet ./...` | Static analysis |
| Format | `cd ov && gofmt -w .` | Format code |

## Project Directory Structure

```
project/
├── bin/ov                     # Built by `task build:ov` (gitignored)
├── ov/                        # Go module (go 1.25.3, kong CLI, go-containerregistry)
├── build.yml                  # Unified build-time config: distro: bootstrap + formats,
│                              # builder: multi-stage defs, init: supervisord/systemd
│                              # (referenced via image.yml format_config)
├── .build/                    # Generated Containerfiles (gitignored)
├── image.yml                  # Image definitions
├── setup.sh                   # Bootstrap: downloads task, builds ov
├── Taskfile.yml               # Bootstrap tasks only
├── taskfiles/                 # Build.yml, Setup.yml
├── layers/<name>/             # Layer directories (160 layers)
├── plugins/                   # Git submodule (overthink-plugins, 5 plugins, 242 skills)
└── templates/                 # supervisord.header.conf (referenced by init.supervisord.header_file)
```

Submodule convention: `plugins/` is a submodule rooted at the
`overthink-plugins` repo. Clone with `--recurse-submodules` or run
`git submodule update --init` after a plain clone. See
`/ov-dev:skills` for the skill-authoring and sync conventions.

## Source Code Map

### Core Generation

| File | Purpose |
|------|---------|
| `main.go` | CLI entry point (Kong framework) |
| `config.go` | `image.yml` parsing, inheritance resolution. `BuildFormats` type. `Distro` field. `ResolvedImage.Tags` (union). `SupportsTag()`, `SupportsBuild()` methods |
| `format_config.go` | `DistroConfig` (with per-distro `Formats`), `BuilderConfig` types. `BuildFile` loader struct matches the three top-level sections of `build.yml` (`distro:`, `builder:`, `init:`). `LoadBuildConfigForImage` resolves a single `format_config: build.yml` ref and splits it into `DistroConfig` / `BuilderConfig` / `InitConfig` views. Per-image config resolution with remote ref support |
| `format_template.go` | Go `text/template` rendering engine. Template helpers: `cacheMounts`, `cacheMountsOwned`, `quote`, `default`, `splitFirst`, `replace`, `join`. `InstallContext`, `BuildStageContext` types |
| `layers.go` | Layer scanning, file detection, `parseLayerYAML()`. `LayerYAML` struct with `Vars` + `Tasks` fields. `Task` struct + `Kind()` method (exactly-one-verb). `PackageSection` generic format sections. `TagSections` for distro overrides. |
| `tasks.go` | **All task emission logic** — per-verb emitters (`emitMkdirBatch`, `emitCopy`, `emitWrite`, `emitLinkBatch`, `emitDownload`, `emitSetcapBatch`, `emitCmd`, `emitBuild`), `emitTasks` orchestrator, `stageInlineContent` (content-addressed), `resolveUserSpec`, `taskSubstPath`, `taskUnresolvedRefs`. Adjacent-coalescing (`taskCoalescesWith`). **Shell-quoting helpers:** `shellSingleQuote(s)` for standard `'...'` escaping (used by LABEL values + `emitDownload` env entries) and `shellAnsiQuote(s)` for bash ANSI-C `$'...'` quoting (used by `emitCmd` so multi-line script bodies survive podman's line-oriented Dockerfile parser). **`emitDownload` env rule:** uses `export VAR=val;` (semicolon-terminated) not `VAR=val cmd`, because bash expands `${VAR}` in URL arguments before the cmd-prefix environment is assembled. ~430 lines, single home for install-task codegen. |
| `generate.go` | Containerfile generation. `writeLayerSteps` orchestrates per-layer: packages → `emitTasks` (from `tasks.go`) → builders → USER reset. Config-driven format install, builder stages, and bootstrap from `build.yml` (`distro:` + `builder:` sections). **`writeLabels` is called at the END of the final stage (after the final `USER` directive)** — the volatile `LabelTests` value means a test edit used to invalidate every downstream RUN/COPY; with LABELs-at-end, only the LABEL steps themselves re-emit (cache preserves all install work). `writeJSONLabel` routes every JSON label value through `shellSingleQuote` so embedded `'` chars in test commands (`awk '{print $1}'`) don't break podman's `key=value` LABEL parser. |
| `validate.go` | All validation rules. `validateLayerTasks` enforces exactly-one-verb, per-verb required modifiers, `vars` key rules, path/mode/caps format, `${VAR}` resolution checks. Format/builder validation against config definitions (not hardcoded maps) |
| `version.go` | CalVer computation |
| `scaffold.go` | `new layer` scaffolding |

### Dependency & Graph

| File | Purpose |
|------|---------|
| `graph.go` | Topological sort (layers + images), `ResolveImageOrder()` |
| `intermediates.go` | Auto-intermediate image computation (trie analysis) |

### Build & Runtime

| File | Purpose |
|------|---------|
| `build.go` | `build` command (sequential image building, retry logic) |
| `merge.go` | `merge` command (post-build layer merging) |
| `shell.go` | `shell` command (execs engine run) |
| `start.go` | `start`/`stop` commands |
| `status.go` | `status` command (structured table/detail view, live tool probing, `--json`) |
| `commands.go` | `enable`/`disable`/`logs`/`update`/`remove` |
| `service.go` | `service` command (init system service management inside containers) |
| `seed.go` | `seed` command (bind-backed volume data seeding) |
| `hooks.go` | Lifecycle hooks (`post_enable`, `pre_remove`) collection and execution |
| `remote_image.go` | Remote image ref resolution, pull-or-build |
| `vm.go` | VM lifecycle: create, start, stop, destroy, list, console, ssh |
| `vm_build.go` | VM disk image builds (qcow2, raw via bootc install) |
| `vm_libvirt.go` | Libvirt backend: VM operations via session-level libvirt |
| `vm_qemu.go` | QEMU backend: direct VM operations via qemu-system |
| `smbios_credentials.go` | SSH key injection via SMBIOS/systemd credentials at VM boot |
| `libvirt.go` | Libvirt XML snippet collection and injection |
| `browser.go` | Browser automation commands (open, list, close, text, html, url, screenshot, click, type, eval, wait, cdp) |
| `browser_cdp.go` | CDPClient -- lightweight Chrome DevTools Protocol WebSocket client (golang.org/x/net/websocket) |
| `cdp.go` | CDP commands + `CdpCmd` struct. `cdpGetWindowOffset`, `cdpDispatchKeyEvent`, `deepQueryJS` |
| `cdp_spa.go` | SPA-aware remote desktop interaction: `CdpSpaCmd` (click, type, key, key-combo, mouse, status). `spaDetect`, `spaEnsureFocus`, `spaKeyMap`, `spaModifierMap`, coordinate scaling via `--scale` |
| `browser_test.go` | Browser command and CDP client tests |
| `wl.go` | Wayland desktop commands (screenshot, click, type, key, mouse, status, windows, focus). `--from-x11` flag + `FindX11WindowGeometry()` for XWayland coordinate translation |
| `vnc.go` | VNC desktop commands (screenshot, click, type, key, mouse, status, passwd, rfb). `--from-x11` flag for XWayland coordinate translation |
| `sway.go` | Sway compositor commands. `swayNode` has `Focused`/`FullscreenMode` fields; `searchSwayNode` prefers focused/fullscreen nodes; XWayland class matching via `swayWindowProperties` |

### Infrastructure

| File | Purpose |
|------|---------|
| `engine.go` | Docker/Podman abstraction, `ResolveImageEngineForDeploy()` |
| `registry.go` | Remote image inspection (go-containerregistry) |
| `transfer.go` | Cross-engine image transfer |
| `runtime_config.go` | `~/.config/ov/config.yml`, `secret_backend` key, credential maps |
| `network.go` | Shared "ov" container network management |
| `machine.go` | Podman machine management (rootful VM builds) |

### Configuration

| File | Purpose |
|------|---------|
| `env.go` | ENV merging, path expansion |
| `envfile.go` | `.env` file parsing (`ParseEnvFile`, `ParseEnvBytes`), runtime env var resolution/merging |
| `security.go` | Container security config collection, CLI args generation. Merges `Mounts` from layer security configs |
| `labels.go` | OCI label constants. `LabelTests` carries the three-section `{layer, image, deploy}` test manifest; `ImageMetadata.Tests *LabelTestSet` is populated by `ExtractMetadata` when present |
| `volumes.go` | Named volume collection/mounting |
| `alias.go` | Command aliases (wrapper scripts) |
| `deploy.go` | Per-deployment config overlay, `DeployVolumeConfig`, `ResolveVolumeBacking()`, `saveDeployState()`, `cleanDeployEntry()` (instance-aware provides cleanup) |
| `provides.go` | Env/MCP provides injection, `removeBySource()`, `removeByExactSource()` (instance-specific cleanup), `podAwareMCPProvides()` |
| `enc.go` | Encrypted volumes (gocryptfs via `systemd-run --scope --user --unit=ov-enc-<image>-<volume>`), `ResolvedBindMount`. `-allow_other` required for rootless podman keep-id. `encUnmount()` stops scope units after fusermount. Stale scope retry on mount failure |
| `devices.go` | Host device auto-detection. `DetectedDevices` struct with `RenderNode` field (first `/dev/dri/renderD*`). `appendAutoDetectedEnv()` centralizes injection of `HSA_OVERRIDE_GFX_VERSION`, `DRINODE`, `DRI_NODE` — called at 10 sites across config_image.go, start.go, shell.go. Uses `appendEnvUnique` so user `-e` flags always override |
| `tunnel.go` | Tunnel providers (Tailscale, Cloudflare), backend scheme helpers (`schemeTarget`, `tailscaleFlag`, `isTCPFamily`), scheme validation maps (`validTailscaleSchemes`, `validCloudflareSchemes`), start/stop for each provider |
| `quadlet.go` | Quadlet .container file generation, `Secret=` directives |
| `credential_store.go` | `CredentialStore` interface, `ResolveCredential()`, `DefaultCredentialStore()`, `ConfigMigrateSecretsCmd` |
| `credential_keyring.go` | System keyring backend (`go-keyring`: GNOME Keyring, KDE Wallet, KeePassXC) |
| `credential_config.go` | Config file credential backend (plaintext fallback for headless) |
| `credential_kdbx.go` | KeePass .kdbx backend (`gokeepasslib`: KDBX 4, Argon2, encrypted at rest) |
| `secrets.go` | Container secret collection from labels, Podman secret provisioning, `SecretArgs()` |
| `secrets_cmd.go` | `ov secrets` CLI commands (init, list, get, set, delete, import, export, path) |
| `secrets_gpg.go` | `ov secrets gpg` commands (show, env, edit, encrypt, decrypt, set, unset, add-recipient, recipients) |

### Remote Layer Refs

| File | Purpose |
|------|---------|
| `refs.go` | Remote ref types, parsing, cache management |
| `refs_git.go` | Git operations: clone, resolve ref, tag resolution |

### Declarative Testing

Implements the `ov test` / `ov image test` commands and the
`org.overthinkos.tests` OCI label. User-facing authoring, verb catalog,
runtime variables, and deploy.yml overlay rules live in `/ov:test` — this
section is the Go-implementation map.

| File | Purpose |
|------|---------|
| `testspec.go` | `Check` struct (19 verb discriminators; `Kind()` enforces exactly-one). Original 15 built-in verbs (file/port/command/http/package/service/process/dns/user/group/interface/kernel-param/mount/addr/matching) plus 4 live-container verbs (`cdp`/`wl`/`dbus`/`vnc`) dispatched via `testrun_ov_verbs.go`. **`Status` on the http verb is a plain `int`** — not a MatcherList. One expected code per test; no `[200, 302]` list shorthand. `Matcher` + `MatcherList` with custom YAML **and** JSON unmarshalers for scalar/list/map shorthand — symmetry between layer.yml authoring and hand-crafted OCI labels. `LabelTestSet` with `{Layer, Image, Deploy}` sections. Extended `${NAME[:arg]}` regex (`testVarRefPattern`) — backward-compatible widening of `taskVarRefPattern` in `tasks.go`. **No bash-style defaults**: `${VAR:-fallback}` is unsupported; only `${IDENT}`. `ExpandTestVars`, `TestVarRefs`, `IsRuntimeOnlyVar`, `Check.ExpandVars`. |
| `testvars.go` | `ResolveTestVarsBuild` / `ResolveTestVarsRuntime`. `InspectContainer` is a swappable package-level `var` (test-friendly pattern matching `InspectLabels` in `labels.go`). Maps `podman inspect` output into `HOST_PORT:<N>`, `VOLUME_PATH:<name>`, `VOLUME_CONTAINER_PATH:<name>`, `CONTAINER_IP`, `CONTAINER_NAME`, `ENV_<NAME>`. |
| `testrun.go` | `Runner`, `Executor` interface, `ContainerExecutor` (via `podman exec`), `ImageExecutor` (via `podman run --rm`). `TestStatus`/`TestResult` types (renamed from `CheckStatus`/`CheckResult` to avoid collision with doctor.go). Per-verb dispatch for `file`/`port`/`command`/`http`. Matcher evaluation: `matchOne` + `matchNumeric` (`lt`/`le`/`gt`/`ge`). `validMatcherOps` allowlist kept in lockstep with the runner switch by `TestMatcher_AllowlistRunnerSync`. Output formatters: text, JSON, TAP. |
| `testrun_verbs.go` | Dispatch for the remaining verbs: `package` (rpm/dpkg/pacman), `service` (supervisorctl + systemctl), `process` (pgrep), `dns` (host-side `net.LookupIP` or in-container `getent`), `user`/`group` (getent passwd/group), `interface` (`ip -o addr show` + MTU), `kernel-param` (`sysctl -n`), `mount` (`findmnt`), `addr` (host-side `net.DialTimeout` or in-container `nc -z`), `matching` (pure in-process value matching). |
| `testrun_ov_verbs.go` | Dispatch for the four live-container verbs (`cdp`/`wl`/`dbus`/`vnc`). Hand-enumerated method allowlists (`cdpMethods`/`wlMethods`/`dbusMethods`/`vncMethods`) map each method name to its `ov test <verb> <method>` subcommand path, required modifier fields, and positional-arg builder. The `runOvVerb` dispatcher handles skip (RunModeImageTest → skip with message; empty `r.Image` → skip), required-modifier enforcement (via `checkRequiredFields` + `isZeroField`), subprocess exec via `findOvBinary()`, matcher pipeline through `matchAll`, and post-run `artifact_min_bytes` size assertions for screenshot methods. |
| `local_image.go` | `resolveLocalImageRef(engine, input)` — test-mode-only image resolution that never reads `image.yml`. Full refs pass through with a `LocalImageExists` check; short names match against `ListLocalImages()` output using label-preferred matching (`org.overthinkos.image=<name>`) with a repo-name trailing-component fallback. Returns `ErrImageNotLocal` on no-match so `FormatCLIError` renders the "ov image pull / ov image build" recommendation. Used by `ImageTestCmd.Run()` to keep `ov image test` purely OCI-labels-driven. |
| `testcollect.go` | `CollectTests(cfg, layers, imageName) *LabelTestSet` walks the base-image chain — mirror of `CollectHooks` in `hooks.go:18-68` — with a visited-image guard so pathological cycles reported by `validateImageDAG` can't hang the collector. Bucketizes checks into `layer`/`image`/`deploy` by source + scope, stamps `Origin` for reporting. `MergeDeployTests(baked, local)` implements id-based replace, append, and `{id: X, skip: true}` disable semantics. |
| `test_cmd.go` | `TestCmd` (top-level `ov test`) and `ImageTestCmd` (`ov image test`) kong command structs. `ov test` flow: `resolveContainer` → `containerImageRef` → `ExtractMetadata` → load `DeployImageConfig.Tests` overlay → `MergeDeployTests` → `ResolveTestVarsRuntime` → populate `Runner.Image`/`Instance` (so `cdp`/`wl`/`dbus`/`vnc` verbs can build CLI invocations) → `Runner.Run` → format results. `ov image test` flow: `resolveLocalImageRef` (never reads image.yml — see `local_image.go`) → `ExtractMetadata` → `ResolveTestVarsBuild` → disposable `ImageExecutor` → run only `layer`+`image` sections (add `--include-deploy` for the deploy section). |
| `validate_tests.go` | `validateTests(cfg, layers, errs)` hooked into `Validate` in `validate.go`. Enforces: exactly-one-verb per Check, attribute types, port range (1-65535), `time.Duration` parse on `timeout`, `scope` ∈ {build,deploy}, build-scope checks can't reference runtime-only variables (via `IsRuntimeOnlyVar`), `id:` uniqueness per section (including cross-layer collisions via `validateCollectedIDUniqueness` → `CollectTests`), matcher-op allowlist (kept in lockstep with `matchOne`), per-verb method-allowlist and required-modifier checks for `cdp`/`wl`/`dbus`/`vnc` (via `validateOvVerb` — deploy-scope-only enforcement, method validation against `cdpMethods`/`wlMethods`/`dbusMethods`/`vncMethods` maps in `testrun_ov_verbs.go`). |

**Related skill**: `/ov:test` is the authoring-facing reference.

## Go Module Info

- Go version: 1.25.3
- Key dependencies: `kong` (CLI), `go-containerregistry` (OCI), `go-keyring` (Secret Service API), `gokeepasslib` (KeePass .kdbx)
- Module path: `ov/go.mod`

## Common Workflows

### Add a New CLI Command

1. Define command struct in appropriate file (or new file)
2. Add to CLI struct in `main.go`
3. Implement `Run()` method
4. Add tests in `*_test.go`
5. Build and test: `cd ov && go test ./... && go build -o ../bin/ov .`

### Add a New Validation Rule

Add to `ov/validate.go`. All validation rules are centralized there.

### Debug a Build Issue

```bash
# Generate Containerfiles without building
bin/ov image generate

# Inspect generated output
cat .build/<image>/Containerfile

# Validate configuration
bin/ov image validate

# Inspect resolved image config
bin/ov image inspect <image>
```

### Intermediate image cache invalidation

`ov image build` auto-generates intermediate images (e.g., `ghcr.io/overthinkos/fedora-ov-2-dbus-nodejs`) that bundle the `ov` layer plus common layers for cache reuse across many downstream images. These intermediates are aggressively podman-cached. Updating `layers/ov/bin/ov` does invalidate the COPY step inside the intermediate, but if the intermediate tag already exists locally, `ov image build` may reuse it without re-running the build chain. To force a fresh binary propagation after a manual `bin/ov` update:

```bash
podman rmi 'ghcr.io/overthinkos/fedora-ov-2*' 2>/dev/null || true
ov image build <image>
```

This also interacts with the dual-path gotcha documented in `/ov-layers:ov`: `bin/ov` (repo-root, used by host-side invocations) and `layers/ov/bin/ov` (what the `ov` layer actually copies into images) must stay in sync. The canonical `task build:ov` path does both; a manual `go build -o bin/ov ./ov` needs an explicit `cp bin/ov layers/ov/bin/ov` follow-up.

## Style Guide

- All logic belongs in Go. Taskfiles are only for bootstrap (building ov).
- Taskfiles for bootstrap only, Go for all other logic.
- Test files alongside source files (`foo.go` -> `foo_test.go`).

## Cross-References

- `/ov-dev:generate` — Understanding generated Containerfiles + deep dive on the task emission pipeline (`ov/tasks.go`).
- `/ov:layer` — **Canonical author-facing reference** for the task verb catalog that `ov/tasks.go` implements.
- `/ov:validate` — Validation rules and error handling (`validateLayerTasks` in `ov/validate.go`).
- `/ov:build` — Using the built CLI.
- `/ov:test` — Author-facing reference for the declarative-testing feature that `testspec.go` / `testvars.go` / `testrun.go` / `testrun_verbs.go` / `testrun_ov_verbs.go` / `testcollect.go` / `test_cmd.go` / `local_image.go` / `validate_tests.go` implement.
- Source: `ov/` directory (~77 source + ~54 test .go files).

## When to Use This Skill

**MUST be invoked** before reading or modifying Go source files. Invoke this skill BEFORE launching Explore agents on ov/ code.
