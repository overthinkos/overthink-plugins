---
name: validate
description: |
  Validation: ov validate command, validation rules, common errors.
  Use when checking images.yml and layer definitions for correctness.
---

# Validate - Validation Commands

## Overview

`ov validate` checks `images.yml` and all layer definitions for errors. Validation collects all errors at once rather than failing on the first.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Validate all | `ov validate` | Check images.yml + all layers |
| Check version | `ov version` | Verify CalVer computation |
| Inspect image | `ov inspect <image>` | Show resolved config |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Validation passed |
| 1 | Validation or user error |
| 2 | Internal error |

## Validation Rules

### Layer Rules

- Layer directory must contain at least one install file
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

### Bind Mount Rules

- `name` and `path` required, name must match volume name regex
- No duplicate names within an image
- Encrypted: `host` must be empty; Plain: `host` required
- Names matching layer volume names override the volume (note, not error)
- Warning if `gocryptfs` not in PATH when encrypted mounts exist

### Tunnel Rules

- Tailscale serve: HTTPS port must be in allowed set (80, 443, 3000-10000, 4443, 5432, 6443, 8443)
- Tailscale funnel: HTTPS must be 443, 8443, or 10000
- Cloudflare: `fqdn` required

### Module Rules

- Two remote modules exporting the same layer name is an error
- Local layers shadow remote layers with same name (note emitted)

## Common Validation Errors

### "circular dependency"

Layers form a dependency cycle. Check `depends` fields.

### "layer X not found"

A `depends` entry or `images.yml` layer references a non-existent layer.

### "PATH must not be set directly"

Use `path_append` in layer.yml instead of `env: PATH: ...`.

### "duplicate volume name"

Volume names must be unique within a layer and across bind mounts.

## Common Workflows

### Validate Before Building

```bash
ov validate && task build:local -- my-image
```

### Debug Validation Errors

```bash
ov validate 2>&1                     # See all errors at once
ov inspect <image>                   # Check resolved config
ov list layers                       # Verify layer exists
```

## Cross-References

- `/overthink:layer` -- Layer definition rules
- `/overthink:image` -- Image configuration rules
- `/overthink:build` -- Building validated images

## When to Use This Skill

Use when the user asks about:

- Running `ov validate`
- Understanding validation errors
- Layer or image definition correctness
- "Why does validation fail?"
