---
name: validate
description: |
  MUST be invoked before any work involving: charly box validate command, validation rules, common validation errors, or checking charly.yml and layer definitions.
---

# charly box validate -- Validation Commands

Invoked as `charly box validate`. See `/charly-image:image` for the family overview.

## Overview

`charly box validate` checks `charly.yml` and all layer definitions for errors. Validation collects all errors at once rather than failing on the first.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Validate all | `charly box validate` | Check charly.yml + all layers (enabled images only) |
| Validate disabled too | `charly box validate --include-disabled` | Extend the validation pass to images marked `enabled: false` (does not modify charly.yml) |
| Check version | `charly version` | Verify CalVer computation |
| Inspect image | `charly box inspect <image>` | Show resolved config |

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Validation passed |
| 1 | Validation or user error |
| 2 | Internal error |

## Validation Rules

### Layer Rules

- Layer directory must contain at least one install source — `charly.yml` with a non-empty `task:` list or a `rpm:` / `deb:` / `pac:` / `aur:` packages section; an auto-detected builder manifest (`pixi.toml`, `pyproject.toml`, `environment.yml`, `package.json`, `Cargo.toml`); or a `candy:` composition field (pure composition layers are valid).
- `depends` must reference existing layers (local or remote).
- Circular dependencies are errors.
- `volumes` names must match `^[a-z0-9]+(-[a-z0-9]+)*$`.
- Volume names must be unique within a candy.
- `aliases` in charly.yml require both `name` and `command`.
- Alias names must match `^[a-zA-Z0-9][a-zA-Z0-9._-]*$`.
- Setting `PATH` directly in `env` is an error (use `path_append`).
- Only one pixi manifest per layer (`pixi.toml`, `pyproject.toml`, or `environment.yml`).

### `task:` Rules

See `/charly-image:layer` for the full verb catalog. The validator enforces:

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

- `base` must reference a valid external image or another image in charly.yml
- `layers` field is required
- `builder` must reference an existing image
- Self-referencing builder is allowed (skipped by generator)
- `bootc: true` requires appropriate base image
- Duplicate alias names within an image are errors
- Box-level alias `command` defaults to `name` if omitted
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
  `charly box reconcile` to align the pins and clear any warning. A fetched layer
  with no `version:` IS a hard error (the layer kind requires one).
- Different layers from the same repo can use different versions
- Collection is reachability-scoped: only layers reachable from the enabled
  images' `base:`/`builder:` chains are fetched. See `/charly-internals:go`
  "Remote-layer resolver" and `/charly-build:reconcile`.

## Common Validation Errors

### "circular dependency"

Layers form a dependency cycle. Check `depends` fields.

### "layer X not found"

A `depends` entry or `charly.yml` layer references a non-existent layer.

### "PATH must not be set directly"

Use `path_append` in charly.yml instead of `env: PATH: ...`.

### "duplicate volume name"

Volume names must be unique within a candy.

## Common Workflows

### Validate Before Building

```bash
charly box validate && charly box build my-image
```

### Debug Validation Errors

```bash
charly box validate 2>&1                     # See all errors at once
charly box inspect <image>                   # Check resolved config
charly box list candies                       # Verify layer exists
```

## Project directory override

`charly box validate` resolves `charly.yml` via `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `CHARLY_PROJECT_DIR=<dir>`. See `/charly-image:image` "Project directory resolution".

## Cross-References

### `charly box` family siblings

- `/charly-image:image` -- Family overview + charly.yml composition reference
- `/charly-build:build` -- Building validated images
- `/charly-build:generate` -- Containerfile generation after validation
- `/charly-build:inspect` -- Inspect a specific image after validation
- `/charly-build:list` -- Enumerate images/layers to validate
- `/charly-build:merge` -- Post-build layer consolidation
- `/charly-build:new` -- Scaffold new candies before validation
- `/charly-build:pull` -- Pull prebuilt images (orthogonal to validation)

### Related skills

- `/charly-image:layer` — **Canonical reference** for the task verb catalog, `var:` substitution, YAML anchors, execution order. The validator rules above enforce what's documented there.
- `/charly-build:generate` — What the generator emits from validated input (per-verb emitters, cache-mount inheritance, inline-content staging).
- `/charly-internals:generate-source` — Internal architecture of the task emission pipeline.
- `/charly-eval:eval` — `charly box validate` schema-checks every `eval:` entry: exactly-one-verb, attribute types, scope/variable consistency (build-scope can't reference runtime-only vars), `id:` uniqueness per section, matcher operator allowlist, unroutable-check rejection. The five live-container verbs (`cdp`/`wl`/`dbus`/`vnc`/`mcp`) also get per-verb method-allowlist + required-modifier enforcement via `validateCharlyVerb` (deploy-scope-only; unknown methods rejected with the allowed set listed).
- `/charly-build:charly-mcp-cmd` — the standalone reference for the `mcp:` verb: required modifiers (`tool:` for `call`, `uri:` for `read`), the 7-method allowlist, and the URL-rewrite / port-publishing behavior that authors occasionally hit.
- `/charly-eval:cdp`, `/charly-eval:wl`, `/charly-eval:dbus`, `/charly-eval:vnc` — per-verb references for the other four live-container verbs.

## Cross-kind name reuse — NOT a uniqueness violation

`charly box validate` does NOT enforce global name uniqueness across kinds. The same name MAY exist simultaneously as a layer (`candy/<name>/`), a `box:` entry, a `pod:` entry, a `vm:` entry, a `k8s:` entry, a `local:` entry, AND a `deploy:` entry. Uniqueness is scoped to each kind. Do not write a validator that flags `box.foo + vm.foo` as ambiguous — verbs disambiguate by command context. The loader raises hard load-time errors on: (a) the obsolete `deploy.qc` / `deploy.cachyos-dx` keys; (b) any obsolete `kind: deployment` doc or root-key `deployment:` (the deploy kind is `kind: deploy`). Every such error points at `charly migrate`. See CLAUDE.md "Cross-kind name reuse is permitted and encouraged" and `/charly-build:migrate`.

## When to Use This Skill

**MUST be invoked** when the task involves charly box validate command, validation rules, common validation errors, or checking charly.yml and layer definitions. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-build. Validate before building to catch errors early.
