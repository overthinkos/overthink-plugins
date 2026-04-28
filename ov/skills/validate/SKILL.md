---
name: validate
description: |
  MUST be invoked before any work involving: ov image validate command, validation rules, common validation errors, or checking image.yml and layer definitions.
---

# ov image validate -- Validation Commands

Invoked as `ov image validate`. See `/ov:image` for the family overview.

## Overview

`ov image validate` checks `image.yml` and all layer definitions for errors. Validation collects all errors at once rather than failing on the first.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Validate all | `ov image validate` | Check image.yml + all layers |
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

- Layer directory must contain at least one install source — `layer.yml` with a non-empty `tasks:` list or a `rpm:` / `deb:` / `pac:` / `aur:` packages section; an auto-detected builder manifest (`pixi.toml`, `pyproject.toml`, `environment.yml`, `package.json`, `Cargo.toml`); or a `layers:` composition field (pure composition layers are valid).
- `depends` must reference existing layers (local or remote).
- Circular dependencies are errors.
- `volumes` names must match `^[a-z0-9]+(-[a-z0-9]+)*$`.
- Volume names must be unique within a layer.
- `aliases` in layer.yml require both `name` and `command`.
- Alias names must match `^[a-zA-Z0-9][a-zA-Z0-9._-]*$`.
- Setting `PATH` directly in `env` is an error (use `path_append`).
- Only one pixi manifest per layer (`pixi.toml`, `pyproject.toml`, or `environment.yml`).

### `tasks:` Rules

See `/ov:layer` for the full verb catalog. The validator enforces:

- **Exactly one verb per task.** A task map must contain exactly one of `cmd` / `mkdir` / `copy` / `write` / `link` / `download` / `setcap` / `build`. Zero verbs → `"task has no action"`; multiple → `"task has conflicting actions: X and Y"`.
- **Per-verb required modifiers:**
  - `copy` → `to:` (destination) required; `copy:` value must be relative to the layer directory, no `..` traversal
  - `write` → `content:` required (non-empty)
  - `link` → `target:` required (what the symlink points to)
  - `download` → `to:` required unless `extract: sh` (piped install scripts)
  - `setcap` with non-empty `caps:` → caps pattern check (`cap_name=flags[,cap_name=flags]`)
- **Path rules:** paths must be absolute, `~/`-prefixed, or `${HOME}`-prefixed. `copy:` source must exist under the layer directory at generate time.
- **Mode rules:** `mode:` must match `^0[0-7]{3,4}$` (octal) if present.
- **Download extract values:** must be one of `tar.gz` / `tar.xz` / `tar.zst` / `zip` / `none` / `sh` or empty.
- **`build:` value:** must be `"all"` (initial implementation; specific builder names reserved for future use).
- **`user:` format:** must be `root`, `${USER}`, a literal name matching `^[a-z_][a-z0-9_-]*$`, or numeric `<uid>:<gid>`. Unresolved `${VAR}` in `user:` errors.

### `vars:` Rules

- Keys must match `^[A-Z_][A-Z0-9_]*$` (shell identifier).
- Keys may not collide with reserved auto-exports (`USER`, `UID`, `GID`, `HOME`, `ARCH`, `BUILD_ARCH`).
- Keys may not collide with the same layer's `env:` keys.

### `${VAR}` Reference Resolution

- In non-shell fields (paths, URLs, `to`, `target`, etc.), every `${NAME}` reference must resolve against `vars:` ∪ auto-exports. Unresolved references error at validate time.
- In shell fields (`cmd:` values, `write: content:`), references are passed through verbatim and resolved by bash at build time.

### Image Rules

- `base` must reference a valid external image or another image in image.yml
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

A `depends` entry or `image.yml` layer references a non-existent layer.

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

## Project directory override

`ov image validate` resolves `image.yml` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. See `/ov:image` "Project directory resolution".

## Cross-References

### `ov image` family siblings

- `/ov:image` -- Family overview + image.yml composition reference
- `/ov:build` -- Building validated images
- `/ov:generate` -- Containerfile generation after validation
- `/ov:inspect` -- Inspect a specific image after validation
- `/ov:list` -- Enumerate images/layers to validate
- `/ov:merge` -- Post-build layer consolidation
- `/ov:new` -- Scaffold new layers before validation
- `/ov:pull` -- Pull prebuilt images (orthogonal to validation)

### Related skills

- `/ov:layer` — **Canonical reference** for the task verb catalog, `vars:` substitution, YAML anchors, execution order. The validator rules above enforce what's documented there.
- `/ov:generate` — What the generator emits from validated input (per-verb emitters, cache-mount inheritance, inline-content staging).
- `/ov-dev:generate` — Internal architecture of the task emission pipeline.
- `/ov:eval` — `ov image validate` schema-checks every `tests:` entry: exactly-one-verb, attribute types, scope/variable consistency (build-scope can't reference runtime-only vars), `id:` uniqueness per section, matcher operator allowlist, unroutable-check rejection. The five live-container verbs (`cdp`/`wl`/`dbus`/`vnc`/`mcp`) also get per-verb method-allowlist + required-modifier enforcement via `validateOvVerb` (deploy-scope-only; unknown methods rejected with the allowed set listed).
- `/ov:mcp` — the standalone reference for the `mcp:` verb: required modifiers (`tool:` for `call`, `uri:` for `read`), the 7-method allowlist, and the URL-rewrite / port-publishing behavior that authors occasionally hit.
- `/ov:cdp`, `/ov:wl`, `/ov:dbus`, `/ov:vnc` — per-verb references for the other four live-container verbs.

## When to Use This Skill

**MUST be invoked** when the task involves ov image validate command, validation rules, common validation errors, or checking image.yml and layer definitions. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Validate before building to catch errors early.
