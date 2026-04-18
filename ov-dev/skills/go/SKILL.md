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

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build | `task build:ov` | Compile to `bin/ov` and install as Arch package |
| Install | `task build:install` | Install ov as Arch package (uses pre-built binary) |
| Run tests | `cd ov && go test ./...` | Run all tests |
| Run specific test | `cd ov && go test -run TestName ./...` | Run single test |
| Vet | `cd ov && go vet ./...` | Static analysis |
| Format | `cd ov && gofmt -w .` | Format code |

## Source Code Map

### Core Generation

| File | Purpose |
|------|---------|
| `main.go` | CLI entry point (Kong framework) |
| `config.go` | `image.yml` parsing, inheritance resolution. `BuildFormats` type. `Distro` field. `ResolvedImage.Tags` (union). `SupportsTag()`, `SupportsBuild()` methods |
| `format_config.go` | `DistroConfig` (with per-distro `Formats`), `BuilderConfig` types. `BuildFile` loader struct matches the three top-level sections of `build.yml` (`distro:`, `builder:`, `init:`). `LoadBuildConfigForImage` resolves a single `format_config: build.yml` ref and splits it into `DistroConfig` / `BuilderConfig` / `InitConfig` views. Per-image config resolution with remote ref support |
| `format_template.go` | Go `text/template` rendering engine. Template helpers: `cacheMounts`, `cacheMountsOwned`, `quote`, `default`, `splitFirst`, `replace`, `join`. `InstallContext`, `BuildStageContext` types |
| `layers.go` | Layer scanning, file detection, `parseLayerYAML()`. `LayerYAML` struct with `Vars` + `Tasks` fields. `Task` struct + `Kind()` method (exactly-one-verb). `PackageSection` generic format sections. `TagSections` for distro overrides. |
| `tasks.go` | **All task emission logic** — per-verb emitters (`emitMkdirBatch`, `emitCopy`, `emitWrite`, `emitLinkBatch`, `emitDownload`, `emitSetcapBatch`, `emitCmd`, `emitBuild`), `emitTasks` orchestrator, `stageInlineContent` (content-addressed), `resolveUserSpec`, `taskSubstPath`, `taskUnresolvedRefs`. Adjacent-coalescing (`taskCoalescesWith`). ~380 lines, single home for install-task codegen. |
| `generate.go` | Containerfile generation. `writeLayerSteps` orchestrates per-layer: packages → `emitTasks` (from `tasks.go`) → builders → USER reset. Config-driven format install, builder stages, and bootstrap from `build.yml` (`distro:` + `builder:` sections). |
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
| `labels.go` | OCI label constants |
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

### Testing

All `*_test.go` files provide tests for their corresponding source files.

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

## Style Guide

- All logic belongs in Go. Taskfiles are only for bootstrap (building ov).
- Taskfiles for bootstrap only, Go for all other logic.
- Test files alongside source files (`foo.go` -> `foo_test.go`).

## Cross-References

- `/ov-dev:generate` — Understanding generated Containerfiles + deep dive on the task emission pipeline (`ov/tasks.go`).
- `/ov:layer` — **Canonical author-facing reference** for the task verb catalog that `ov/tasks.go` implements.
- `/ov:validate` — Validation rules and error handling (`validateLayerTasks` in `ov/validate.go`).
- `/ov:build` — Using the built CLI.
- Source: `ov/` directory (~68 source + ~48 test .go files).

## When to Use This Skill

**MUST be invoked** before reading or modifying Go source files. Invoke this skill BEFORE launching Explore agents on ov/ code.
