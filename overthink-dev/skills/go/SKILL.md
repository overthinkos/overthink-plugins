---
name: go
description: |
  Go CLI development: building the ov binary, running tests, understanding
  the source code structure. Use when working on the ov/ Go module.
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
| `config.go` | `images.yml` parsing, inheritance resolution |
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
| `commands.go` | `enable`/`disable`/`status`/`logs`/`update`/`remove` |
| `seed.go` | `seed` command (bind mount data seeding) |
| `remote_image.go` | Remote image ref resolution, pull-or-build |
| `vm.go` | VM lifecycle: create, start, stop, destroy, list, console, ssh |
| `vm_build.go` | VM disk image builds (qcow2, raw via bcvk) |
| `libvirt.go` | Libvirt XML snippet collection and injection |

### Infrastructure

| File | Purpose |
|------|---------|
| `engine.go` | Docker/Podman abstraction |
| `registry.go` | Remote image inspection (go-containerregistry) |
| `transfer.go` | Cross-engine image transfer |
| `runtime_config.go` | `~/.config/ov/config.yml` |

### Configuration

| File | Purpose |
|------|---------|
| `env.go` | ENV merging, path expansion |
| `envfile.go` | `.env` file parsing, runtime env var resolution/merging |
| `security.go` | Container security config collection, CLI args generation |
| `labels.go` | OCI label constants |
| `volumes.go` | Named volume collection/mounting |
| `alias.go` | Command aliases (wrapper scripts) |
| `deploy.go` | Per-deployment config overlay |
| `crypto.go` | Encrypted bind mounts (gocryptfs) |
| `gpu.go` | GPU detection |
| `tunnel.go` | Tunnel configuration (Tailscale, Cloudflare) |
| `quadlet.go` | Quadlet .container file generation |

### Modules

| File | Purpose |
|------|---------|
| `mod.go` | Remote module types, parsing |
| `mod_cmd.go` | CLI commands: mod get/download/tidy/verify/update/list |
| `mod_git.go` | Git operations: clone, resolve ref, compute hash |

### Testing

All `*_test.go` files provide tests for their corresponding source files.

## Go Module Info

- Go version: 1.25.3
- Key dependencies: `kong` v1.14.0 (CLI), `go-containerregistry` v0.20.7 (OCI)
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

- `/overthink-dev:generate` -- Understanding generated Containerfiles
- `/overthink:validate` -- Validation rules and error handling
- `/overthink:build` -- Using the built CLI
- Source: `ov/` directory (64 .go files)

## When to Use This Skill

Use when the user asks about:

- Building the ov CLI from source
- Running Go tests
- Understanding the Go code structure
- Adding new CLI commands or features
- Debugging Go code
- "Where is the code for X?"
