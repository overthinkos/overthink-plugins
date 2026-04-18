---
name: validate
description: |
  MUST be invoked before any work involving: ov image validate command, validation rules, common validation errors, or checking images.yml and layer definitions.
---

# ov image validate -- Validation Commands

Invoked as `ov image validate`. See `/ov:image` for the family overview.

## Overview

`ov image validate` checks `images.yml` and all layer definitions for errors. Validation collects all errors at once rather than failing on the first.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Validate all | `ov image validate` | Check images.yml + all layers |
| Check version | `ov version` | Verify CalVer computation |
| Inspect image | `ov image inspect <image>` | Show resolved config |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Validation passed |
| 1 | Validation or user error |
| 2 | Internal error |

## Validation Rules

### Layer Rules

- Layer directory must contain at least one install file (`layer.yml` rpm/deb, `root.yml`, `pixi.toml`, `pyproject.toml`, `environment.yml`, `package.json`, `Cargo.toml`, or `user.yml`) **or** a `layers:` field in `layer.yml` (pure composition)
- `depends` must reference existing layers (local or remote)
- Circular dependencies are errors
- `volumes` names must match `^[a-z0-9]+(-[a-z0-9]+)*$`
- Volume names must be unique within a layer
- `aliases` in layer.yml require both `name` and `command`
- Alias names must match `^[a-zA-Z0-9][a-zA-Z0-9._-]*$`
- Setting `PATH` directly in `env` is an error (use `path_append`)
- Only one pixi manifest per layer (`pixi.toml`, `pyproject.toml`, or `environment.yml`)

### Image Rules

- `base` must reference a valid external image or another image in images.yml
- `layers` field is required
- `builder` must reference an existing image
- Self-referencing builder is allowed (skipped by generator)
- `bootc: true` requires appropriate base image
- Duplicate alias names within an image are errors
- Image-level alias `command` defaults to `name` if omitted
- `env` accepts list of `KEY=VALUE` strings (runtime only)
- `env_file` accepts a path string (validated at runtime)
- `security` at image level overrides layer-level security (not an error)

### Bind Mount Rules

- `name` and `path` required, name must match volume name regex
- No duplicate names within an image
- Encrypted: `host` must be empty; Plain: `host` required
- Names matching layer volume names override the volume (note, not error)
- Warning if `gocryptfs` not in PATH when encrypted mounts exist

### Tunnel Rules

- Tailscale serve (tailnet-only): any port works for HTTPS
- Tailscale funnel (public): HTTPS port must be 443, 8443, or 10000
- Cloudflare: `fqdn` required

### VM / Bootc Rules

- `bootc: true` requires appropriate base image
- `vm.rootfs` must be `ext4`, `xfs`, or `btrfs`
- `vm.backend` must be `auto`, `libvirt`, or `qemu`

### Port Relay Rules

- Each relay port must be 1-65535
- Each relay port must also be declared in the layer's `ports:`
- No duplicate relay ports across layers in the same image

### Tunnel Multi-Port Rules

- `ports` must be `"all"` or omitted
- When `ports: all`, the image must have at least one port defined
- Multi-port mode skips single-port HTTPS validation

### Port Protocol Rules

- Only `tcp:` prefix is supported
- Port number after prefix must be 1-65535

### Remote Layer Rules

- Two remote repos exporting the same layer name is an error
- Local layers shadow remote layers with same name (note emitted)
- Same bare ref at conflicting versions is an error
- Different layers from the same repo can use different versions

## Common Validation Errors

### "circular dependency"

Layers form a dependency cycle. Check `depends` fields.

### "layer X not found"

A `depends` entry or `images.yml` layer references a non-existent layer.

### "PATH must not be set directly"

Use `path_append` in layer.yml instead of `env: PATH: ...`.

### "duplicate volume name"

Volume names must be unique within a layer.

## Common Workflows

### Validate Before Building

```bash
ov image validate && ov image build my-image
```

### Debug Validation Errors

```bash
ov image validate 2>&1                     # See all errors at once
ov image inspect <image>                   # Check resolved config
ov image list layers                       # Verify layer exists
```

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + images.yml composition reference
- `/ov:build` -- Building validated images
- `/ov:generate` -- Containerfile generation after validation
- `/ov:inspect` -- Inspect a specific image after validation
- `/ov:list` -- Enumerate images/layers to validate
- `/ov:merge` -- Post-build layer consolidation
- `/ov:new` -- Scaffold new layers before validation
- `/ov:pull` -- Pull prebuilt images (orthogonal to validation)

### Related skills

- `/ov:layer` -- Layer definition rules

## When to Use This Skill

**MUST be invoked** when the task involves ov image validate command, validation rules, common validation errors, or checking images.yml and layer definitions. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Validate before building to catch errors early.
