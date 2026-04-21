---
name: install-plan
description: |
  The InstallPlan IR — the shared intermediate representation consumed by build-mode
  Containerfile emission (OCITarget), container deploys (ContainerDeployTarget), and
  host deploys (HostDeployTarget).
  MUST be invoked before reading or modifying any of:
  ov/install_plan.go, ov/install_build.go, ov/build_target_oci.go,
  ov/deploy_target_host.go, ov/deploy_target_container.go, or when adding
  a new step kind / deploy target / reverse-op kind.
---

# InstallPlan IR — shared IR for build + deploy

## Overview

`ov` has three code paths that all need to know "what does applying this layer mean?":

1. **Build mode** — `ov image build` / `ov image generate` emit Containerfiles.
2. **Container deploy** — `ov deploy add <name> <ref>` runs the image via quadlet, optionally building an overlay image when `add_layers:` is set.
3. **Host deploy** — `ov deploy add host <ref>` applies the recipe to the invoking user's filesystem.

Before the 2026-04 refactor these were three separate walks over `Layer` / `ResolvedImage`. The refactor unifies them behind one IR. A pure compiler (`BuildDeployPlan`) turns `Layer + ResolvedImage + HostContext` into an `InstallPlan`; each code path becomes a `DeployTarget` consuming the plan.

This skill is the single source of truth for the IR shape. Add a new step kind by editing `install_plan.go`, the compiler in `install_build.go`, and each target's `emit*` method — this skill lists every place that needs to stay in sync.

## File layout

| File | Role |
|---|---|
| `ov/install_plan.go` | IR types, enums, `InstallStep` interface, `ReverseOp`, `DeployTarget` interface, `EmitOpts`, `GateEnabled` |
| `ov/install_build.go` | `BuildDeployPlan` pure compiler: `Layer` → `InstallPlan` |
| `ov/build_target_oci.go` | `OCITarget` — emits Containerfile text |
| `ov/deploy_target_container.go` | `ContainerDeployTarget` — synthesizes overlay Containerfile when `add_layers:` present, delegates to quadlet/start |
| `ov/deploy_target_host.go` | `HostDeployTarget` — executes plan on host via shell + `podman run <builder>` |
| `ov/install_plan_test.go` | IR unit tests (scope/venue/gate/reverse derivations) |
| `ov/install_build_test.go` | Compiler integration tests (ripgrep, dev-tools, pixi layer) |

## Core types

### `InstallPlan`

```go
type InstallPlan struct {
    DeployID       string       // per-deploy hash of image + add_layers
    Image          string
    Version        string       // layer/image CalVer
    Distro         string       // "fedora:43"
    Layer          string       // set for per-layer plans; "" for merged whole-image plans
    Steps          []InstallStep
    LayersIncluded []string     // ordered topo-sorted layer names (for merged plans)
    AddLayers      []string     // refs added via deploy.yml add_layers: (for provenance)
    BuilderImage   string       // selected builder for VenueContainerBuilder steps
    Meta           map[string]string
}
```

One plan per layer when the compiler runs on a single `Layer`. For whole-image deploys, `MergePlans(plans, image, addLayers)` merges per-layer plans while preserving layer boundaries for refcount bookkeeping. `DeployID` is a deterministic 16-hex-char sha256 prefix over `(image, layer_order, add_layers)` — same inputs → same ID, so re-deploys are stable.

### `InstallStep` interface

```go
type InstallStep interface {
    Kind() StepKind
    Scope() Scope
    Venue() Venue
    RequiresGate() Gate
    Reverse() []ReverseOp
}
```

All eight concrete step kinds implement this interface. `Reverse()` is called at install time (not teardown) so the ledger records the exact reversal ops tied to the specific artifacts created.

### Enums

**`Scope`** — where the effect lands:
- `ScopeSystem` — `/etc`, `/usr`, `/var`; requires sudo on host; `USER root` in Containerfile.
- `ScopeUser` — `$HOME/.pixi`, `$HOME/.cargo`, `$HOME/.local`, user-scope systemd units.
- `ScopeUserProfile` — shell init surface (`~/.bashrc` / `~/.zshenv` / fish conf.d + `~/.config/overthink/env.d/`).

**`Venue`** — where commands physically execute:
- `VenueHostNative` — host shell (plain or sudo-wrapped) or a plain `RUN` in Containerfile.
- `VenueContainerBuilder` — inside a builder container: `FROM <builder> AS stage` in Containerfile; `podman run <builder>` on host target.
- `VenueSkip` — recorded with reason but not executed (container-runtime-only fields on host target; `aur:` on non-Arch hosts).

**`Phase`** — three-phase template execution:
- `PhasePrepare` — repo config, key import, copr enable.
- `PhaseInstall` — the actual package-manager or builder invocation.
- `PhaseCleanup` — teardown (copr disable, cache wipe).

Each `SystemPackagesStep` carries one phase; `--allow-repo-changes` gating is a simple lookup on `step.Phase == PhasePrepare`.

**`Gate`** — opt-in flag name:
- `GateNone` (default, always enabled)
- `GateAllowRepoChanges`
- `GateAllowRootTasks`
- `GateWithServices`

Gates apply only to the host target. `EmitOpts.AssumeYes` enables all three. See `GateEnabled(gate, opts)` in `install_plan.go`.

**`StepKind`** — discriminator for concrete types:
- `SystemPackages`, `Builder`, `Task`, `File`, `ServicePackaged`, `ServiceCustom`, `ShellHook`, `RepoChange`.

## The eight step kinds

| Kind | What it carries | Venue default | Scope derivation |
|---|---|---|---|
| `SystemPackagesStep` | Format (rpm/deb/pac), Phase, Packages, Repos, Options, Copr, Modules, Exclude, Keys, CacheMounts | HostNative | Always system |
| `BuilderStep` | Builder (pixi/npm/cargo/aur), BuilderImage, LayerDir, Phase, Artifacts, RawStageContext | ContainerBuilder | aur→system, others→user |
| `TaskStep` | Task (raw), LayerName, LayerDir, CtxPath, ResolvedUser | HostNative | From ResolvedUser (root or 0 → system; else user) |
| `FileStep` | Source, Dest, Mode, Owner, LayerName | HostNative | `pathIsSystemScoped(Dest)` |
| `ServicePackagedStep` | Unit, TargetScope, Enable, OverridesText, OverridesPath, LayerName, PriorEnabled | HostNative | TargetScope field |
| `ServiceCustomStep` | Name, UnitText, UnitPath, TargetScope, Enable, LayerName | HostNative | TargetScope field |
| `ShellHookStep` | LayerName, EnvVars, PathAdd, EnvFile | HostNative | Always user-profile |
| `RepoChangeStep` | Format, File, Content, Checksum, LayerName | HostNative | Always system |

Each step's `Reverse()` emits typed `ReverseOp` values. Adding a step kind means: (a) define the struct in `install_plan.go`, (b) decide its Scope/Venue/Gate/Reverse, (c) add a case to each target's step dispatch (`emit*` in OCITarget; `exec*` in HostDeployTarget), (d) ensure the compiler in `install_build.go` emits it.

## The compiler — `BuildDeployPlan`

```go
func BuildDeployPlan(layer *Layer, img *ResolvedImage, hostCtx HostContext) (*InstallPlan, error)
```

Pure — no I/O, no side effects. Given the same inputs, produces the same plan. Called:
- Once per layer during `ov image build` (OCITarget walks the combined output).
- Once per layer during `ov deploy add <container>` (ContainerDeployTarget filters to `add_layers:` for overlay synthesis).
- Once per layer during `ov deploy add host` (HostDeployTarget walks the combined output).

Pass `HostContext{Target: "host", Distro: ..., GlibcVersion: ...}` for host compilation; zero-value for build/container compilation.

Step emission order (mirrors today's `writeLayerSteps`):
1. `ShellHookStep` for `env:` + `path_append:` (deterministic map ordering).
2. `SystemPackagesStep`(s) — distro-tag section wins over build-format section (first-match precedence from `ResolvedImage.Distro` order).
3. `TaskStep`(s) in YAML order.
4. `BuilderStep`(s) for each matching multi-stage or inline builder.
5. `ServicePackagedStep` / `ServiceCustomStep` from unified `services:` (legacy `service:`/`system_services:` fall through `compileServiceStepsLegacy`).

`MergePlans([]*InstallPlan, image, addLayers)` composes per-layer plans into a single whole-image plan for target-level walking (sudo batching, single dry-run output).

## The `DeployTarget` interface

```go
type DeployTarget interface {
    Name() string                 // "oci" | "container" | "host"
    Emit(plans []*InstallPlan, opts EmitOpts) error
}
```

Three implementations:

### `OCITarget` (`ov/build_target_oci.go`)
Emits Containerfile text. Consumes `phases.install.container` from build.yml (falls back to `install_template:`). For multi-stage builders, delegates to `Generator.buildStageContext` for the existing `BuildStageContext` template rendering. For tasks, delegates to `Generator.emitTasks` with a temporary layer-tasks swap so the existing per-verb emitters (`emitCopy`, `emitWrite`, `emitCmd`, `emitMkdirBatch`, ...) run unchanged.

Used by: `ov image build`, `ov image generate`, `ov deploy add <container>` (overlay Containerfile synthesis).

### `ContainerDeployTarget` (`ov/deploy_target_container.go`)
Wraps the existing quadlet/podman pipeline with overlay-Containerfile synthesis for `add_layers:`. Picks an overlay tag deterministically from `(base-image, sorted-layer-set)`. Removed on `ov deploy del` unless `--keep-image`.

Used by: `ov deploy add <container>` with `add_layers:` present.

### `HostDeployTarget` (`ov/deploy_target_host.go`)
Walks the IR; groups contiguous same-`(Scope, Venue)` steps via `plan.StepsByVenue()`; emits one heredoc per batch. Full executor: writes service units (packaged + custom), env.d files, managed blocks, ledger entries. Invokes builder containers via `builder_run.go` for `VenueContainerBuilder` steps. Gates (`GateEnabled`) applied per step; skipped steps logged.

See `/ov-dev:host-infra` for supporting files (hostdistro, ledger, reverse_ops, shell_profile, builder_run, service_render, deploy_ref).

## `StepBatch` batching

`InstallPlan.StepsByVenue()` partitions `Steps` into contiguous same-`(Scope, Venue)` runs. HostDeployTarget uses this to emit one shell heredoc per batch:

| Batch | Emission form |
|---|---|
| `{ScopeSystem, VenueHostNative}` | `sudo bash <<'OV_ROOT' … OV_ROOT` |
| `{ScopeUser, VenueHostNative}` | `bash <<'OV_USER' … OV_USER` |
| `{_, VenueContainerBuilder}` | `podman run <builder> bash -s < script` |
| `{_, VenueSkip}` | Logged, no exec |

OCITarget doesn't batch — it emits each step in order as Containerfile directives; adjacent same-`USER` steps collapse naturally via the existing `USER` switching logic in `emitTasks`.

## `ReverseOp` catalogue

See `/ov:host-deploy` for the user-facing reverse-op table. The Go-level source of truth is `ReverseOpKind` in `install_plan.go`; each step's `Reverse()` method emits ops tagged with kind + targets + scope. Execution lives in `ov/reverse_ops.go` — one handler per kind, all routed through `runReverseOps(ops, executor)` in LIFO order.

Adding a new reverse kind requires:
1. Add the `ReverseOpKind` constant in `install_plan.go`.
2. Emit it from the appropriate step's `Reverse()` method.
3. Add a handler function in `reverse_ops.go` and register it in `runReverseOp`'s dispatch switch.

## `EmitOpts` cross-cutting flags

```go
type EmitOpts struct {
    DryRun               bool
    FormatJSON           bool
    AllowRepoChanges     bool
    AllowRootTasks       bool
    WithServices         bool
    SkipIncompatible     bool
    AssumeYes            bool
    Verify               bool
    Pull                 bool
    BuilderImageOverride string
}
```

CLI flags on `DeployAddCmd` / `DeployDelCmd` populate this struct; each target reads what it needs. `AssumeYes` enables all three opt-in gates (via `GateEnabled`).

## Testing

- `install_plan_test.go` — 13 unit tests over step-kind derivations (scope/venue/gate/reverse).
- `install_build_test.go` — 8 integration tests that load real `layers/` via `ScanAllLayersWithConfig`; `testHostContextWithDistro` helper.
- `install_build.go:200` comment — canonical fixture docs.

When you add a step kind, add:
1. A scope/venue/gate/reverse unit test in `install_plan_test.go`.
2. An integration test in `install_build_test.go` exercising the compiler path.
3. Target-specific tests in `build_target_oci_test.go` and `deploy_target_host_test.go`.

## Cross-References

- `/ov-dev:host-infra` — supporting files (hostdistro, ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref)
- `/ov-dev:go` — overall Go code map; Kong CLI framework; mode-purity invariant
- `/ov-dev:generate` — Containerfile generation call graph; how OCITarget plugs into Generator
- `/ov:deploy` — user-facing `ov deploy add`/`del` surface
- `/ov:host-deploy` — host-target user-facing behavior (ledger, gates, ReverseOps)
- `/ov:build` — build-mode user-facing surface; three-phase template story
- `/ov:layer` — layer.yml schema including unified `services:` that map to `ServicePackagedStep` / `ServiceCustomStep`

## When to Use This Skill

**MUST be invoked** when reading or modifying any of `ov/install_plan.go`, `ov/install_build.go`, `ov/build_target_*.go`, `ov/deploy_target_*.go`; when adding a new step kind, target, or reverse-op kind; or when debugging why a particular layer produces a plan that doesn't match expectations. Invoke BEFORE reading the source files or running Explore agents against this subsystem.
