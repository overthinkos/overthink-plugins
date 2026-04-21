---
name: host-infra
description: |
  Host-deploy supporting Go files: hostdistro.go, install_ledger.go,
  builder_run.go, shell_profile.go, reverse_ops.go, service_render.go,
  deploy_ref.go, migrate_services_tool.go.
  MUST be invoked before reading or modifying any of those files, or when
  debugging host-target deploy behaviour (ledger state, sudo batching,
  managed-block insertion, glibc preflight, ref resolution).
---

# Host Infrastructure — Go files that support `HostDeployTarget`

## Overview

The InstallPlan IR (see `/ov-dev:install-plan`) is the central data type. `HostDeployTarget` (the skill you want for the executor itself) consumes the IR and invokes a handful of supporting files for each concrete concern: host distro detection, ledger I/O, builder-container invocation, shell profile writes, ReverseOp execution, service rendering, and ref resolution. This skill is the one-per-file reference.

## File map

| File | Purpose | Key exports |
|---|---|---|
| `ov/hostdistro.go` | Detect host distro from `/etc/os-release`; detect host glibc via `ldd --version` | `HostDistro`, `DetectHostDistro`, `DetectHostGlibc`, `CompareGlibc`, `distroIDAliases` |
| `ov/install_ledger.go` | Flock-serialized JSON ledger at `~/.config/overthink/installed/` | `LedgerPaths`, `LedgerLock`, `DeployRecord`, `LayerRecord`, `StepRecord`, `AcquireLedgerLock`, `AddLayerDeployment`, `RemoveLayerDeployment` |
| `ov/builder_run.go` | `podman run <builder>` wrapper with HOME-remap + bind-mount helpers | `BuilderRun`, `BuilderRunOpts`, `UserScopeBindMounts`, `UserScopeEnv` |
| `ov/shell_profile.go` | bash/zsh/fish detection + managed-block fencing + env.d I/O | `ShellKind`, `DetectLoginShell`, `EnvdDir`, `WriteEnvdFile`, `RemoveEnvdFile`, `EnsureManagedBlock`, `RemoveManagedBlock`, `ShellInitFilePath` |
| `ov/reverse_ops.go` | Execute `ReverseOp` slices in LIFO order via per-kind handlers | `runReverseOps`, `ReverseExecutor` interface, 15 reverse handlers |
| `ov/service_render.go` | Render `ServiceEntry` → supervisord INI / systemd unit via init-system template | `ServiceEntry`, `ServiceOverrides`, `RenderService`, `RenderedService`, `ServiceRenderContext`, template helpers (`systemdRestart`, `supervisordRestart`, `systemdStdout`, `supervisordLog`) |
| `ov/deploy_ref.go` | Unified 4-form ref resolver (local name / local path / remote github / legacy `@host/...`) | `DeployRef`, `RefKind`, `RefSource`, `ResolveDeployRef`, `classifyYAMLFile` |
| `ov/migrate_services_tool.go` | One-shot migration from legacy `service:`/`system_services:` → unified `services:` (kept for external layer sources) | `MigrateServicesDir`, `migrateLayerFile` |

## `hostdistro.go` — distro detection

`DetectHostDistro()` reads `/etc/os-release` line-by-line, parses `ID=`, `VERSION_ID=`, `ID_LIKE=`, and populates an ordered `Tags` slice for layer tag-section matching.

### Distro ID aliases

Critical: `/etc/os-release` `ID=` values don't always match `build.yml` distro keys. Arch reports `ID=arch` but build.yml keys it as `archlinux`; RHEL-family distros (almalinux/rocky/centos/rhel) map to `fedora`. The alias table:

| os-release ID | build.yml key |
|---|---|
| `arch`, `archarm`, `manjaro`, `endeavouros` | `archlinux` |
| `almalinux`, `rocky`, `centos`, `rhel` | `fedora` |

`HostDistro.populateTags()` appends both the os-release name AND the canonical build.yml key so layer tag sections (`arch:`, `fedora:`) and `DistroConfig.ResolveDistro` lookups both succeed.

### Glibc preflight

`DetectHostGlibc()` runs `ldd --version`, parses the trailing `MAJOR.MINOR` via regex (`glibcRegexp`). Returns `""` with no error for non-glibc systems (musl/alpine) — callers treat empty as "unknown, skip preflight".

`CompareGlibc(a, b)` returns -1/0/+1; `""` on either side returns 0 (unknown is compatible). The builder-glibc-greater-than-host case is refused by `HostDeployTarget` before any container runs.

### `FormatHint()`

Convenience for the compiler when no `ResolvedImage` is available (single-layer ad-hoc deploys). Maps `arch*` → `pac`, `fedora`/`rhel`/`centos` → `rpm`, `debian`/`ubuntu` → `deb`.

## `install_ledger.go` — persistent deploy records

Layout at `~/.config/overthink/installed/`:

```
.lock                  flock for concurrent `ov deploy` sessions
deploys/<id>.json      per-deploy record
layers/<name>.json     per-layer record (refcount source of truth)
```

### `DeployRecord`

```go
type DeployRecord struct {
    DeployID   string
    Image      string
    Tag        string
    Target     string   // "host" or "container:<name>"
    Layers     []string
    AddLayers  []string
    DeployedAt string
}
```

### `LayerRecord`

```go
type LayerRecord struct {
    Layer        string
    Version      string
    DeployedBy   []string  // set of deploy IDs — refcount
    DeployedAt   string
    BuilderImage string
    Steps        []StepRecord
    ReverseOps   []ReverseOp
}
```

`DeployedBy` is the refcount source. `AddLayerDeployment(paths, layer, deployID, update)` appends to the set idempotently; `RemoveLayerDeployment(paths, layer, deployID)` decrements, returning `(record, shouldFullyRemove)` where `shouldFullyRemove == true` means the set just drained and reverse ops should run.

### Flock protocol

`AcquireLedgerLock(paths)` opens `.lock` with `O_RDWR|O_CREATE` and takes an exclusive flock. `Release()` unlocks + closes. A second concurrent `ov deploy` blocks on the lock. This is the sole serialization primitive; step writes are not individually locked because the session holds the whole-directory lock.

Writes are atomic via `writeJSONAtomic(path, data)` — JSON to `path.tmp` then rename.

## `builder_run.go` — podman invocation for VenueContainerBuilder

`BuilderRun(ctx, opts)` wraps `exec.CommandContext("podman", ...)` with the argv assembly in `buildBuilderRunArgs`:

- `run --rm --user $(id -u):$(id -g) -i`
- Bind-mounts in deterministic order (sorted by container path) so dry-run output is reproducible
- Layer source mounted read-only at `/work`
- `-e HOME=<host home>`, plus builder-specific env (`PIXI_CACHE_DIR`, etc.)
- `-w /work`
- Image ref, `bash -s` (script on stdin)

`BuilderRunOpts.DryRun` short-circuits to log-only output on stderr.

### Standard bind-mount helpers

`UserScopeBindMounts(hostHome)` returns the standard subdir set:
- `$HOME/.pixi`, `$HOME/.cargo`, `$HOME/.npm-global`, `$HOME/.cache/ov`

Ensures the dirs exist on host (empty dirs are fine; podman would reject non-existent source paths otherwise).

`UserScopeEnv(hostHome)` returns the env-var set pixi/npm/cargo need to honor the bind-mounted paths rather than the builder's own home:
- `PIXI_CACHE_DIR=$HOME/.cache/ov/pixi`
- `RATTLER_CACHE_DIR=$HOME/.cache/ov/rattler`
- `NPM_CONFIG_PREFIX=$HOME/.npm-global`
- `CARGO_HOME=$HOME/.cargo`

## `shell_profile.go` — bash/zsh/fish integration

### Login shell detection

`DetectLoginShell()` inspects `$SHELL`; falls back to `/etc/passwd` via `os/user`. Returns `ShellBash` / `ShellZsh` / `ShellFish`; unknown defaults to `ShellBash`.

### env.d files

Each host-deployed layer gets `~/.config/overthink/env.d/<layer>.env` via `WriteEnvdFile(home, layer, envVars, pathAdd)`. Contents rendered deterministically (sorted env keys; deterministic PATH join):

```sh
# overthink env for layer pre-commit — managed by ov; do not edit
export PIXI_CACHE_DIR="$HOME/.cache/pixi"
export PATH=/home/user/.pixi/envs/default/bin:$PATH
```

`RemoveEnvdFile(home, layer)` deletes the file; not-found is silently OK.

### Managed-block fencing

`EnsureManagedBlock(shell, home)` inserts (or updates idempotently) a fenced block in the shell's init file:

```sh
# overthink:begin (managed by ov; do not edit inside this block)
for f in "$HOME/.config/overthink/env.d"/*.env; do [ -r "$f" ] && . "$f"; done
# overthink:end
```

Target per shell via `ShellInitFilePath(shell, home)`:
- bash → `~/.profile`
- zsh → `~/.zshenv`
- fish → `~/.config/fish/conf.d/overthink.fish` (fish syntax; uses `for f in <glob>; source $f; end`)

Implementation: `replaceOrAppendManagedBlock(existing, body)` edits in place; `stripManagedBlock(existing)` removes. Both preserve surrounding user content.

`RemoveManagedBlock` is called at the end of `ov deploy del host` when no layers remain deployed.

## `reverse_ops.go` — executing the 15 ReverseOp kinds

### `ReverseExecutor` interface

```go
type ReverseExecutor interface {
    reverseDryRun() bool
    reverseKeepRepoChanges() bool
    reverseKeepServices() bool
}
```

`DeployDelCmd` satisfies this via thin wrappers over its `--dry-run`/`--keep-repo-changes`/`--keep-services` flags.

### `runReverseOps(ops, executor)`

Executes ops in **LIFO order** — teardown mirrors install order. Per-op failures log to stderr but don't abort the sequence (partial teardown is better than half-removed; user gets to see which specific op failed).

### Handler dispatch

`runReverseOp` switches on `ReverseOpKind` and dispatches to per-kind handlers:

- `reversePackageRemove` — `sudo dnf/apt-get/pacman remove` (format-dependent argv)
- `reversePixiEnvRemove` — `rm -rf $HOME/.pixi/envs/<name>` directly (no pixi binary needed)
- `reverseCargoUninstall` — `cargo uninstall <bin>` as invoking user
- `reverseNpmUninstallG` — `npm uninstall -g <pkg>`
- `reverseRmFileSystem` — `sudo rm -f <path>`
- `reverseRmFileUser` — `os.Remove(<path>)` (no sudo)
- `reverseRmDir` — `os.RemoveAll(<path>)` / `sudo rm -rf` per scope
- `reverseServiceDisable` — `systemctl [--user] disable --now <unit>` (skipped with `--keep-services`)
- `reverseServiceRemove` — delete unit file (skipped with `--keep-services`)
- `reverseRemoveDropin` — delete drop-in + `rmdir --ignore-fail-on-non-empty` the `.d` parent
- `reverseRestoreEnabled` — re-enable a packaged unit if `PriorEnabled` was true
- `reverseRemoveManaged` — no-op at the per-op level (session-level cleanup in `runHostDel`)
- `reverseRemoveEnvdFile` — `os.Remove(<path>)`
- `reverseRemoveRepoFile` — `sudo rm -f <path>` (skipped with `--keep-repo-changes`)
- `reverseCoprDisable` — `sudo dnf -y copr disable <repo>` (skipped with `--keep-repo-changes`)

### `runSudoArgvReverse(argv, re)`

Sudo wrapper that peels env-var prefixes (e.g., `DEBIAN_FRONTEND=noninteractive`) and re-applies them via `sudo env K=V ... cmd`.

## `service_render.go` — abstract spec → init-native unit text

### `ServiceEntry` struct

Mirrors the `services:` schema in layer.yml (see `/ov:layer`). Fields: `Name`, `UsePackaged`, `Exec`, `Env`, `Restart`, `WorkingDirectory`, `User`, `After`, `Before`, `Stdout`, `StopTimeout`, `Scope`, `Enable`, `Overrides`.

### `RenderService(entry, initDef, ctx)`

Returns `*RenderedService` with `UnitText`/`UnitPath` (custom entries) or `DropinText`/`DropinPath` (packaged entries with overrides). Dispatches on:

- `entry.IsPackaged()` + `initDef.ServiceSchema.SupportsPackaged` — render drop-in via `dropin_template` + `dropin_path_template`.
- `entry.IsPackaged()` + !SupportsPackaged — return error (supervisord can't consume systemd units).
- Custom entry + service_template present — render unit via `service_template` + `unit_path_template`.

### `ServiceRenderContext` fields

The template context exposed to build.yml's init service_template. Includes pre-computed `EnvList []KeyValue` (sorted for deterministic rendering), `SystemUnitDir`, `UserUnitDir`, `FragmentDir`, `PackagedUnit`.

### Template helper funcs

`serviceRenderFuncs()` provides:
- `join`, `systemdRestart`, `supervisordRestart`, `systemdStdout`, `supervisordLog`.

Each init system (`supervisord`, `systemd`) references these funcs from its `service_template` in `build.yml`.

## `deploy_ref.go` — unified 4-form ref resolver

`ResolveDeployRef(ref, projectDir)` returns a `*DeployRef` with `Kind` (image/layer) and `Source` (local-name / local-path / remote). The four accepted ref forms:

1. **Remote legacy** — `@github.com/owner/repo/path:version` — dispatched to `resolveRemoteRef` via `ParseRemoteRef`.
2. **Remote new** — `github.com/owner/repo[/images/<n>|/layers/<n>][@version]` — `looksLikeRemoteRef` + `translateAtVersion` normalize to legacy form.
3. **Local YAML path** — starts with `./`, `../`, `/`, or ends with `.yml`/`.yaml` — `resolveLocalPath` stats file + calls `classifyYAMLFile`.
4. **Local name** — anything else — `resolveLocalName` checks `image.yml` → `layers/`; same name in both is a hard error.

Image vs layer disambiguation:
- Refs containing `/layers/` → layer.
- Refs containing `/images/` → image.
- Bare remote repos (`github.com/owner/repo`) → image (falls back to image.yml).
- Local YAML path: `classifyYAMLFile` reads top-level keys — `images:`/`base:`/`defaults:` → image; `rpm:`/`deb:`/`pac:`/`aur:`/`tasks:`/`services:`/`service:`/`system_services:`/`layers:`/`depends:`/`env:`/`path_append:`/`description:` → layer.

## `migrate_services_tool.go` — one-shot layer migration

`MigrateServicesDir(dir)` walks a `layers/` directory, rewrites each `layer.yml` with legacy `service:`/`system_services:` fields to the unified `services:` list, and returns the count of modified files.

Already used to migrate all 40 in-tree layers (see git history). Kept in-tree for external layer sources that haven't yet migrated; not wired to a CLI subcommand — invoked via the gated test `TestRunMigrateServices` (set `OV_RUN_MIGRATION=1`).

Internals: `extractBlock` parses the scalar `|`-block; `extractListBlock` parses the sequence; `parseSupervisordPrograms` walks `[program:X]` sections; `buildServicesEntries` synthesizes the structured entries; `renderServicesYAML` emits the new block. `yamlScalar` handles quoting for values with YAML indicator characters (notably `%` for supervisord's `%(ENV_HOME)s` interpolations).

## Cross-References

- `/ov-dev:install-plan` — the IR these files support; full step-kind catalogue
- `/ov-dev:go` — overall Go code map; Kong framework; mode-purity invariant
- `/ov-dev:generate` — Containerfile generation call graph
- `/ov:host-deploy` — user-facing host-deploy surface (ledger, gates, ReverseOps)
- `/ov:deploy` — command family overview
- `/ov:layer` — unified `services:` schema authored by layer authors

## When to Use This Skill

**MUST be invoked** when reading or modifying any of the listed files; when debugging host-deploy ledger state; when adding a new `ReverseOpKind` handler, a new shell to the detection map, or a new distro ID alias; or when extending the ref resolver to accept a new ref form.
