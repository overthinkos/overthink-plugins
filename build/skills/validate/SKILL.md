---
name: validate
description: |
  MUST be invoked before any work involving: ov box validate command, validation rules, common validation errors, or checking box.yml and layer definitions.
---

# ov box validate -- Validation Commands

Invoked as `ov box validate`. See `/ov-image:image` for the family overview.

## Overview

`ov box validate` checks `box.yml` and all layer definitions for errors. Validation collects all errors at once rather than failing on the first.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Validate all | `ov box validate` | Check box.yml + all layers (enabled images only) |
| Validate disabled too | `ov box validate --include-disabled` | Extend the validation pass to images marked `enabled: false` (does not modify box.yml) |
| Check version | `ov version` | Verify CalVer computation |
| Inspect image | `ov box inspect <image>` | Show resolved config |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Validation passed |
| 1 | Validation or user error |
| 2 | Internal error |

## Validation Rules

### Layer Rules

- Layer directory must contain at least one install source — `candy.yml` with a non-empty `task:` list or a `rpm:` / `deb:` / `pac:` / `aur:` packages section; an auto-detected builder manifest (`pixi.toml`, `pyproject.toml`, `environment.yml`, `package.json`, `Cargo.toml`); or a `layer:` composition field (pure composition layers are valid).
- `depends` must reference existing layers (local or remote).
- Circular dependencies are errors.
- `volumes` names must match `^[a-z0-9]+(-[a-z0-9]+)*$`.
- Volume names must be unique within a layer.
- `aliases` in candy.yml require both `name` and `command`.
- Alias names must match `^[a-zA-Z0-9][a-zA-Z0-9._-]*$`.
- Setting `PATH` directly in `env` is an error (use `path_append`).
- Only one pixi manifest per layer (`pixi.toml`, `pyproject.toml`, or `environment.yml`).

### `task:` Rules

See `/ov-image:layer` for the full verb catalog. The validator enforces:

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
- **`cache:` rules:** only valid on `cmd:` / `download:` tasks; each path must be absolute (or `~/` / `${HOME}`-prefixed). Declares extra BuildKit cache mounts (owned per the task `user:`).
- **`build:` value:** must be `"all"` (initial implementation; specific builder names reserved for future use).
- **`user:` format:** must be `root`, `${USER}`, a literal name matching `^[a-z_][a-z0-9_-]*$`, or numeric `<uid>:<gid>`. Unresolved `${VAR}` in `user:` errors.

### `var:` Rules

- Keys must match `^[A-Z_][A-Z0-9_]*$` (shell identifier).
- Keys may not collide with reserved auto-exports (`USER`, `UID`, `GID`, `HOME`, `ARCH`, `BUILD_ARCH`).
- Keys may not collide with the same layer's `env:` keys.

### `${VAR}` Reference Resolution

- In non-shell fields (paths, URLs, `to`, `target`, etc.), every `${NAME}` reference must resolve against `var:` ∪ auto-exports. Unresolved references error at validate time.
- In shell fields (`cmd:` values, `write: content:`), references are passed through verbatim and resolved by bash at build time.

### Image Rules

- `base` must reference a valid external image or another image in box.yml
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
- Each relay port must also be declared in the layer's `port:`
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
- A layer (bare `@github` ref) referenced via two different git tags is **not**
  an error — the git tag is only the FETCH coordinate. The resolver reads each
  fetched layer's OWN `version:` and dedups by it: same per-entity version → no
  warning (a re-tag of an unchanged layer is silent); different per-entity
  versions → warns once and resolves to the **newest** (highest CalVer). Run
  `ov box reconcile` to align the pins and clear any warning. A fetched layer
  with no `version:` IS a hard error (the layer kind requires one).
- Different layers from the same repo can use different versions
- Collection is reachability-scoped: only layers reachable from the enabled
  images' `base:`/`builder:` chains are fetched. See `/ov-internals:go`
  "Remote-layer resolver" and `/ov-build:reconcile`.

## Common Validation Errors

### "circular dependency"

Layers form a dependency cycle. Check `depends` fields.

### "layer X not found"

A `depends` entry or `box.yml` layer references a non-existent layer.

### "PATH must not be set directly"

Use `path_append` in candy.yml instead of `env: PATH: ...`.

### "duplicate volume name"

Volume names must be unique within a layer.

## Common Workflows

### Validate Before Building

```bash
ov box validate && ov box build my-image
```

### Debug Validation Errors

```bash
ov box validate 2>&1                     # See all errors at once
ov box inspect <image>                   # Check resolved config
ov box list candies                       # Verify layer exists
```

## Project directory override

`ov box validate` resolves `box.yml` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. See `/ov-image:image` "Project directory resolution".

## Cross-References

### `ov image` family siblings

- `/ov-image:image` -- Family overview + box.yml composition reference
- `/ov-build:build` -- Building validated images
- `/ov-build:generate` -- Containerfile generation after validation
- `/ov-build:inspect` -- Inspect a specific image after validation
- `/ov-build:list` -- Enumerate images/layers to validate
- `/ov-build:merge` -- Post-build layer consolidation
- `/ov-build:new` -- Scaffold new layers before validation
- `/ov-build:pull` -- Pull prebuilt images (orthogonal to validation)

### Related skills

- `/ov-image:layer` — **Canonical reference** for the task verb catalog, `var:` substitution, YAML anchors, execution order. The validator rules above enforce what's documented there.
- `/ov-build:generate` — What the generator emits from validated input (per-verb emitters, cache-mount inheritance, inline-content staging).
- `/ov-internals:generate-source` — Internal architecture of the task emission pipeline.
- `/ov-eval:eval` — `ov box validate` schema-checks every `eval:` entry: exactly-one-verb, attribute types, scope/variable consistency (build-scope can't reference runtime-only vars), `id:` uniqueness per section, matcher operator allowlist, unroutable-check rejection. The five live-container verbs (`cdp`/`wl`/`dbus`/`vnc`/`mcp`) also get per-verb method-allowlist + required-modifier enforcement via `validateOvVerb` (deploy-scope-only; unknown methods rejected with the allowed set listed).
- `/ov-build:ov-mcp-cmd` — the standalone reference for the `mcp:` verb: required modifiers (`tool:` for `call`, `uri:` for `read`), the 7-method allowlist, and the URL-rewrite / port-publishing behavior that authors occasionally hit.
- `/ov-eval:cdp`, `/ov-eval:wl`, `/ov-eval:dbus`, `/ov-eval:vnc` — per-verb references for the other four live-container verbs.

## Cross-kind name reuse — NOT a uniqueness violation

`ov box validate` does NOT enforce global name uniqueness across kinds. The same name MAY exist simultaneously as a layer (`candy/<name>/`), an `image:` entry, a `pod:` entry, a `vm:` entry, a `k8s:` entry, a `local:` entry, AND a `deploy:` entry. Uniqueness is scoped to each kind. Do not write a validator that flags `image.foo + vm.foo` as ambiguous — verbs disambiguate by command context. The loader raises hard load-time errors on: (a) the obsolete `deploy.qc` / `deploy.cachyos-dx` keys; (b) any obsolete `kind: deployment` doc or root-key `deployment:` (the deploy kind is `kind: deploy`). Every such error points at `ov migrate`. See CLAUDE.md "Cross-kind name reuse is permitted and encouraged" and `/ov-build:migrate`.

## When to Use This Skill

**MUST be invoked** when the task involves ov box validate command, validation rules, common validation errors, or checking box.yml and layer definitions. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Validate before building to catch errors early.
