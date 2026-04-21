---
name: migrate
description: |
  MUST be invoked before any work involving: `ov migrate unified` command, converting legacy image.yml/layer.yml/build.yml into unified overthink.yml, rewriting flat-form layer.yml into kind-keyed form, or migrating legacy service:|...| raw-INI and system_services: entries into the structured service: list schema.
---

# ov migrate -- Unified-schema migration

Invoked as `ov migrate unified`. One-shot converter from legacy scattered config (`image.yml` + `build.yml` + per-layer flat `layer.yml`) into the canonical unified `overthink.yml` format.

The new format is what every other `ov` command reads (`LoadUnified` in `ov/unified.go`). Legacy forms are no longer accepted at runtime — `LoadConfig` errors with a pointer to this command. See `/ov:layer` (authoring reference) and `/ov:image` (umbrella) for the format documentation itself.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Convert project | `ov migrate unified` | Emit `overthink.yml` (via `includes:`) next to existing legacy files. Non-destructive by default. |
| Convert + rewrite layers | `ov migrate unified --rewrite-layers` | Also rewrite every `layers/<name>/layer.yml` into kind-keyed form (`layer: {...}`). |
| Monolithic output | `ov migrate unified --monolithic` | Emit a single flat `overthink.yml` instead of using `includes:` to reference existing files. |
| Preview | `ov migrate unified --dry-run` | Print what would be written; touch nothing. Combines with `--rewrite-layers` and `--monolithic`. |

## What it converts

### 1. Project structure

`image.yml` + `build.yml` at the repo root → `overthink.yml` with `includes:` referencing them (or a monolithic flattened body with `--monolithic`). Each entry becomes kind-keyed: `build:`, `image:`, `layer:` — matching the types that `LoadUnified` expects.

### 2. Flat-form `layer.yml` → kind-keyed `layer:` wrapper

Legacy flat form:

```yaml
name: redis
depends: [base]
packages: [redis]
```

New form (required — the flat form is rejected by `parseLayerYAML` after migration with a hard error pointing to `--rewrite-layers`):

```yaml
layer:
  name: redis
  depends: [base]
  packages: [redis]
```

Run with `--rewrite-layers` to convert every `layers/*/layer.yml` in the project in one pass.

### 3. Raw-INI `service:` → structured list

Legacy supervisord-style scalar:

```yaml
service: |
  [program:redis]
  command=/usr/bin/redis-server
  autostart=true
  startretries=3
```

Becomes the structured form with all fidelity preserved (including the 8 supervisord directives added to `ServiceEntry` in 2026-04 — `kind`, `events`, `auto_start`, `start_retries`, `start_secs`, `stop_signal`, `exit_codes`, `priority`):

```yaml
service:
  - name: redis
    exec: "/usr/bin/redis-server"
    auto_start: true
    start_retries: 3
    restart: always
```

`[eventlistener:...]` blocks are recognized and emitted with `kind: eventlistener` + `events:` + the original `command:` mapped to `exec:`. The rewrite is driven by `rewriteServiceKeys` in `ov/migrate_unified.go:397`.

### 4. `services:` (plural) → `service:` (singular)

A pure field rename. The unified schema uses the singular `service:` key. External layer repos authored against the plural form are transparently fixed.

### 5. `system_services: [foo, bar]` → structured entries

```yaml
# Before
system_services:
  - sshd
  - cockpit
```

```yaml
# After
service:
  - name: sshd
    use_packaged: sshd.service
  - name: cockpit
    use_packaged: cockpit.service
```

`use_packaged:` tells the generator to reuse the distro-shipped systemd unit rather than render a new one (see `/ov:layer` "Service Declaration" and `/ov-dev:capabilities` for how this flows through the init-system schema in `build.yml`).

## Idempotency

Running `ov migrate unified --rewrite-layers` twice produces byte-identical output (enforced by `TestMigrateUnified_IncludesSplit` and `TestMigrateUnified_Monolithic` in `ov/migrate_unified_test.go`). CI can safely re-run the migrator as a guardrail without churn.

## Auto-invocation on remote-cache downloads

`ov/refs.go:164` invokes `MigrateUnified(... RewriteLayers: true)` automatically whenever a remote `@host/org/repo:version` include lands in the repo cache (`~/.cache/ov/repos/`). This means external projects that still ship the legacy layout pull-through cleanly — no manual step required before the local build can consume the remote layers. See `/ov:pull` for the remote-ref lifecycle.

Downside: the cache directory silently gets rewritten on first fetch. If you want to inspect a remote project's on-disk layout untouched, clone it directly instead of relying on the cache.

## Typical flow

```bash
# Inspect what will change
ov migrate unified --rewrite-layers --dry-run

# Apply the rewrite
ov migrate unified --rewrite-layers

# Confirm the build still works
ov image validate
ov image build <image>
```

After a successful migration, the project's legacy `image.yml` / `build.yml` stay in place (referenced via `includes:` in the new `overthink.yml`). To fully collapse into a single file, re-run with `--monolithic` and then remove the originals.

## See Also

- `/ov:layer` — authoring reference for the `layer:` kind-keyed schema and the full `service:` / 22-field `ServiceEntry`
- `/ov:image` — umbrella skill for `image:` entries and `ov image build/validate/inspect`
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems) that `overthink.yml` references
- `/ov:pull` — remote `@...` refs and the auto-migration hook
- `/ov-dev:install-plan` — the IR that the loader feeds into the build/deploy pipelines
- `/ov-dev:capabilities` — OCI label contract (`LabelServices`) that consumes the migrated service list
- `/ov-dev:go` — loader internals (`LoadUnified`, `parseLayerYAML`, `MigrateUnified`)
