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

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build | `task build:ov` | Compile to `bin/ov` |
| Install | `task build:install` | Build and install to `~/.local/bin` |
| Run tests | `cd ov && go test ./...` | Run all tests |
| Run specific test | `cd ov && go test -run TestName ./...` | Run single test |
| Vet | `cd ov && go vet ./...` | Static analysis |
| Format | `cd ov && gofmt -w .` | Format code |

## Source Code Map

### Core Generation

| File | Purpose |
|------|---------|
| `main.go` | CLI entry point (Kong framework) |
| `config.go` | `images.yml` parsing, inheritance resolution. `SecurityConfig` has `Mounts` field for host mounts (e.g. `/dev/input`, tmpfs) |
| `layers.go` | Layer scanning, file detection, `parseLayerYAML()` |
| `generate.go` | Containerfile generation (largest file) |
| `validate.go` | All validation rules |
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
| `service.go` | `service` command (supervisord service management inside containers) |
| `seed.go` | `seed` command (bind mount data seeding) |
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
| `browser_test.go` | Browser command and CDP client tests |
| `wl.go` | Wayland desktop commands (screenshot, click, type, key, mouse, status, windows, focus). `--from-x11` flag + `FindX11WindowGeometry()` for XWayland coordinate translation |
| `vnc.go` | VNC desktop commands (screenshot, click, type, key, mouse, status, passwd, rfb). `--from-x11` flag for XWayland coordinate translation |
| `sway.go` | Sway compositor commands. `swayNode` has `Focused`/`FullscreenMode` fields; `searchSwayNode` prefers focused/fullscreen nodes; XWayland class matching via `swayWindowProperties` |
| `sun.go` | Sunshine server management commands (status, passwd, pair, clients, config, set, restart, url) + `checkSunStatus()` for tool probing |
| `sun_client.go` | Sunshine REST API client (HTTPS, Basic Auth, port 47990) |
| `sun_test.go` | Sunshine client tests (httptest servers, credential resolution) |
| `moon.go` | GameStream client commands (status, pair, unpair, apps, launch, quit) |
| `moon_client.go` | GameStream protocol client (cert gen, AES pairing crypto, XML parsing, mutual TLS) |
| `moon_test.go` | GameStream client tests (AES roundtrip, cert gen, XML parsing, RSA signing) |

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
| `envfile.go` | `.env` file parsing, runtime env var resolution/merging |
| `security.go` | Container security config collection, CLI args generation. Merges `Mounts` from layer security configs |
| `labels.go` | OCI label constants |
| `volumes.go` | Named volume collection/mounting |
| `alias.go` | Command aliases (wrapper scripts) |
| `deploy.go` | Per-deployment config overlay, `saveDeployState()`, `cleanDeployEntry()` |
| `crypto.go` | Encrypted bind mounts (gocryptfs) |
| `devices.go` | Host device auto-detection (NVIDIA GPU, AMD GPU/KFD, /dev/dri, /dev/kvm, etc.) |
| `tunnel.go` | Tunnel configuration (Tailscale, Cloudflare) |
| `quadlet.go` | Quadlet .container file generation, `Secret=` directives |
| `credential_store.go` | `CredentialStore` interface, `ResolveCredential()`, `DefaultCredentialStore()`, `ConfigMigrateSecretsCmd` |
| `credential_keyring.go` | System keyring backend (`go-keyring`: GNOME Keyring, KDE Wallet, KeePassXC) |
| `credential_config.go` | Config file credential backend (plaintext fallback for headless) |
| `secrets.go` | Container secret collection from labels, Podman secret provisioning, `SecretArgs()` |

### Remote Layer Refs

| File | Purpose |
|------|---------|
| `refs.go` | Remote ref types, parsing, cache management |
| `refs_git.go` | Git operations: clone, resolve ref, tag resolution |

### Testing

All `*_test.go` files provide tests for their corresponding source files.

## Go Module Info

- Go version: 1.25.3
- Key dependencies: `kong` (CLI), `go-containerregistry` (OCI), `go-keyring` (Secret Service API)
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
bin/ov generate

# Inspect generated output
cat .build/<image>/Containerfile

# Validate configuration
bin/ov validate

# Inspect resolved image config
bin/ov inspect <image>
```

## Style Guide

- All logic belongs in Go. Taskfiles are only for bootstrap (building ov).
- Taskfiles for bootstrap only, Go for all other logic.
- Test files alongside source files (`foo.go` -> `foo_test.go`).

## Cross-References

- `/ov-dev:generate` -- Understanding generated Containerfiles
- `/ov:validate` -- Validation rules and error handling
- `/ov:build` -- Using the built CLI
- Source: `ov/` directory (65 source + 46 test .go files)

## When to Use This Skill

**MUST be invoked** before reading or modifying Go source files. Invoke this skill BEFORE launching Explore agents on ov/ code.
