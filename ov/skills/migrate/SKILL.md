---
name: migrate
description: |
  MUST be invoked before any work involving: `ov migrate unified` command (converting legacy image.yml/layer.yml/build.yml into unified overthink.yml, rewriting flat-form layer.yml, migrating legacy service:|...| raw-INI and system_services: entries), or `ov migrate vm-spec` (harvesting legacy image.bootc/image.vm/image.libvirt fields into kind:vm entities in vms.yml).
---

# ov migrate -- Schema migration commands

One-shot, idempotent migrators for every hard-cutover schema change the project has shipped. All `ov migrate <name>` commands follow the same contract (see `/ov-dev:cutover-policy`): running them twice is a no-op, running them once fully transforms the legacy surface into the new one. Legacy forms raise hard load-time errors at runtime — `LoadUnified` / `LoadConfig` errors point at the relevant `ov migrate` sub-verb.

## Sub-verbs

| Sub-verb | Purpose | Skills covering the produced schema |
|---|---|---|
| `ov migrate unified` | Legacy `image.yml` + `build.yml` + flat-form `layer.yml` → `overthink.yml` with kind-keyed wrappers + `includes:` | `/ov:layer`, `/ov:image` |
| `ov migrate vm-spec` | Legacy `image.bootc: true` + `image.vm: {...}` + `image.libvirt: [...]` + layer-level `libvirt:` → `vms.yml` `kind: vm` entities | `/ov-vms:vms`, `/ov-dev:vm-spec` |

# ov migrate unified

Invoked as `ov migrate unified`. One-shot converter from legacy scattered config (`image.yml` + `build.yml` + per-layer flat `layer.yml`) into the canonical unified `overthink.yml` format.

The new format is what every other `ov` command reads (`LoadUnified` in `ov/unified.go`). See `/ov:layer` (authoring reference) and `/ov:image` (umbrella) for the format documentation itself.

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

---

# ov migrate vm-spec

Invoked as `ov migrate vm-spec`. One-shot converter from the legacy per-image VM fields (`image.bootc: true` + `image.vm: {...}` + `image.libvirt: [...]` + layer-level `libvirt:` snippets) into the post-cutover `kind: vm` entity format in `vms.yml`. See `/ov-vms:vms` for the produced schema and `/ov-dev:vm-spec` for the Go types.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Convert project | `ov migrate vm-spec` | Harvest legacy VM fields into `vms.yml`. Idempotent; preserves pre-existing `vms:` keys. |
| Preview | `ov migrate vm-spec --dry-run` | Print what would be written to `vms.yml`; touch nothing. |
| Target file override | `ov migrate vm-spec --output vms-legacy.yml` | Write to a different file instead of `vms.yml`. |

## What it converts

### Per-image fields (kind:image → kind:vm)

| Legacy location | → | New location (vms.yml) |
|---|---|---|
| `images.<name>.bootc: true` | → | `vms.<name>.source.kind: bootc` + `source.image: <name>` |
| `images.<name>.vm.disk_size` | → | `vms.<name>.disk_size` |
| `images.<name>.vm.ram`, `.cpus` | → | `vms.<name>.ram`, `.cpus` |
| `images.<name>.vm.rootfs`, `.root_size`, `.kernel_args` | → | `vms.<name>.source.rootfs`, `.root_size`, `.kernel_args` |
| `images.<name>.vm.ssh_port` | → | `vms.<name>.ssh.port` |
| `images.<name>.vm.firmware` | → | `vms.<name>.firmware` |
| `images.<name>.vm.network` (string) | → | `vms.<name>.network.mode` |
| `images.<name>.vm.transport` | → | `vms.<name>.source.transport` |

### Layer-level libvirt snippets

Layer-level `libvirt: ["<xml>", ...]` on the **contributing layer** is preserved (still supported post-cutover — that's where `/ov-layers:qemu-guest-agent` contributes its virtio-serial channel, for example). Image-level `libvirt: [...]` on the `kind: image` entry is **deleted** — it had no home in the new VM model. The migrator does NOT move image-level libvirt into the VM entity automatically; it emits a warning listing each deleted snippet with a suggestion to re-home it on the produced `vms.<name>.libvirt.snippets:` if still needed.

## Naming convention

Bootc VMs pair 1:1 with their container image. The migrator names produced entries `<image-name>-bootc` or `<image-name>` depending on whether the image name already ends in `-bootc`:

- `image: aurora` (`bootc: true`) → `vms: aurora-bootc`
- `image: selkies-desktop-bootc` (`bootc: true`) → `vms: selkies-desktop-bootc-bootc` (doubled suffix distinguishes VM entity from container image)

## Preservation semantics

**Never clobbers pre-existing `vms:` keys.** If `vms.aurora-bootc:` already exists in `vms.yml`, the migrator skips emission for that key and prints a notice. Lets authors hand-customize entries (adding `libvirt.devices.*`, `cloud_init.packages`, etc.) without losing work on re-run.

**Idempotent.** Running twice produces byte-identical `vms.yml` (enforced by test fixtures in `ov/migrate_vm_spec_test.go`).

## Auto-invocation

Like `ov migrate unified`, `vm-spec` is auto-invoked during remote-cache downloads when `ov/refs.go` detects legacy VM fields in a fetched `@github.com/org/repo:version` include. External repos on the old schema pull through cleanly without a manual migration step.

## Typical flow

```bash
# Inspect what will change
ov migrate vm-spec --dry-run

# Apply the migration
ov migrate vm-spec

# Include the new file from overthink.yml
# (edit overthink.yml to add `vms.yml` under `includes:`)

# Confirm VM builds still work
ov vm build <name>
ov vm create <name>
```

## Post-migration load errors

Once the legacy fields are gone from the schema, old projects loading under the new `ov` binary get hard load errors pointing at this migration:

```
Error: image entry "foo" declares legacy field "bootc: true".
Run: ov migrate vm-spec
```

Remediation hint points at the command directly — no docs reading required.

---

## See Also

- `/ov:layer` — authoring reference for the `layer:` kind-keyed schema and the full `service:` / 22-field `ServiceEntry`
- `/ov:image` — umbrella skill for `image:` entries and `ov image build/validate/inspect`
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems) that `overthink.yml` references
- `/ov:pull` — remote `@...` refs and the auto-migration hook
- `/ov:vm` — `ov vm` command family; reads the `vms.yml` produced by `migrate vm-spec`
- `/ov-vms:vms` — authoring reference for the `kind: vm` entities produced by `migrate vm-spec`
- `/ov-dev:install-plan` — the IR that the loader feeds into the build/deploy pipelines
- `/ov-dev:capabilities` — OCI label contract (`LabelServices`) that consumes the migrated service list
- `/ov-dev:vm-spec` — Go types produced by `migrate vm-spec`
- `/ov-dev:cutover-policy` — policy governing why hard-cutover + idempotent migrator is the required shape
- `/ov-dev:go` — loader internals (`LoadUnified`, `parseLayerYAML`, `MigrateUnified`, `MigrateVmSpec`)
