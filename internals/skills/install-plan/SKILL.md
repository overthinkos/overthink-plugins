---
name: install-plan
description: |
  The InstallPlan IR — the shared intermediate representation consumed by build-mode
  Containerfile emission (OCITarget), pod deploys (PodDeployTarget),
  local deploys (LocalDeployTarget), VM deploys (VmDeployTarget over SSH), and
  Kubernetes deploys (K8sDeployTarget).
  MUST be invoked before reading or modifying any of:
  ov/install_plan.go, ov/install_build.go, ov/build_target_oci.go,
  ov/deploy_target_local.go, ov/deploy_target_pod.go, ov/deploy_target_vm.go,
  ov/k8s_target.go, or when adding
  a new step kind / deploy target / reverse-op kind.
---

# InstallPlan IR — shared IR for build + deploy

## Overview

`charly` has five code paths that all need to know "what does applying this layer mean?":

1. **Build mode** — `charly box build` / `charly box generate` emit Containerfiles.
2. **Pod deploy** (`target: pod`, default) — `charly deploy add <name> <ref>` runs the image via quadlet, optionally building an overlay image when `add_candy:` is set.
3. **Local deploy** (`target: local`) — `charly deploy add <name> <ref> --target local` applies the recipe to the destination machine's filesystem (`host: local` → direct shell; `host: <user@machine>` → SSH).
4. **VM deploy** (`target: vm`) — `charly deploy add vm:<name> <ref>` applies the recipe inside a running VM over SSH.
5. **Kubernetes deploy** (`target: k8s`) — `charly deploy add <name> --target k8s` emits a Kustomize base/overlays tree.

All five paths are unified behind one IR. A pure compiler (`BuildDeployPlan`) turns `Layer + ResolvedImage + HostContext` into an `InstallPlan`; each code path is a `DeployTarget` consuming the plan.

This skill is the single source of truth for the IR shape. Add a new step kind by editing `install_plan.go`, the compiler in `install_build.go`, and each target's `emit*` method — this skill lists every place that needs to stay in sync.

## File layout

| File | Role |
|---|---|
| `ov/install_plan.go` | IR types, enums, `InstallStep` interface, `ReverseOp`, `DeployTarget` interface, `EmitOpts`, `GateEnabled` |
| `ov/install_build.go` | `BuildDeployPlan` pure compiler: `Layer` → `InstallPlan` |
| `ov/build_target_oci.go` | `OCITarget` — emits Containerfile text |
| `ov/deploy_target_pod.go` | `PodDeployTarget` — synthesizes overlay Containerfile when `add_candy:` present, delegates to quadlet/start |
| `ov/deploy_target_local.go` | `LocalDeployTarget` — executes plan on the destination machine via shell + `podman run <builder>` |
| `ov/deploy_target_vm.go` | `VmDeployTarget` — executes plan inside a running VM via SSH + scp |
| `ov/k8s_target.go` | `K8sDeployTarget` — emits Kustomize base/overlays tree |
| `ov/deploy_executor*.go` | `DeployExecutor` interface + `ShellExecutor` + `SSHExecutor` — shell + file-copy abstraction shared by Local/VM targets |
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

All twelve concrete step kinds implement this interface. `Reverse()` is called at install time (not teardown) so the ledger records the exact reversal ops tied to the specific artifacts created.

### Enums

**`Scope`** — where the effect lands:
- `ScopeSystem` — `/etc`, `/usr`, `/var`; requires sudo on host; `USER root` in Containerfile.
- `ScopeUser` — `$HOME/.pixi`, `$HOME/.cargo`, `$HOME/.local`, user-scope systemd units.
- `ScopeUserProfile` — shell init surface (`~/.bashrc` / `~/.zshenv` / fish conf.d + `~/.config/opencharly/env.d/`).

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
- `SystemPackages`, `Builder`, `Task`, `File`, `ServicePackaged`, `ServiceCustom`, `ShellHook`, `ShellSnippet`, `RepoChange`, `ApkInstall`, `LocalPkgInstall`, `Reboot`.

The IR carries no image-fetch step kind. Deploys (any target) emit
zero image-pull / image-build steps; test-bed image preflight is a
separate, eval-time concern handled by `ov/eval_image_preflight.go`
(CLAUDE.md "Deploy fetches NOTHING speculative").

## The twelve step kinds

| Kind | What it carries | Venue default | Scope derivation |
|---|---|---|---|
| `SystemPackagesStep` | Format (rpm/deb/pac), Phase, Packages, Repos, Options, Copr, Modules, Exclude, Keys, CacheMounts | HostNative | Always system |
| `BuilderStep` | Builder (pixi/npm/cargo/aur), BuilderImage, LayerDir, Phase, Artifacts, RawStageContext | ContainerBuilder | aur→system, others→user |
| `TaskStep` | Task (raw), LayerName, LayerDir, CtxPath, ResolvedUser | HostNative | From ResolvedUser (root or 0 → system; else user) |
| `FileStep` | Source, Dest, Mode, Owner, LayerName | HostNative | `pathIsSystemScoped(Dest)` |
| `ServicePackagedStep` | Unit, TargetScope, Enable, OverridesText, OverridesPath, LayerName, PriorEnabled | HostNative | TargetScope field |
| `ServiceCustomStep` | Name, UnitText, UnitPath, TargetScope, Enable, LayerName | HostNative | TargetScope field |
| `ShellHookStep` | LayerName, EnvVars, PathAdd, EnvFile | HostNative | Always user-profile |
| `ShellSnippetStep` | LayerName, Origin, Shell (bash/zsh/fish/sh), Snippet, PathAppend, Destination, Marker, UseDropin, Priority | HostNative | `pathIsSystemScoped(Destination)` (system for container drop-ins, user-profile for ~/.bashrc etc.) |
| `RepoChangeStep` | Format, File, Content, Checksum, LayerName | HostNative | Always system |
| `ApkInstallStep` | Packages (apk specs), LayerName, LayerDir | HostNative | Always system. Only `target: android` executes it; every other target records a skip. |
| `LocalPkgInstallStep` | PkgbuildRef, LayerName, LayerDir, ProjectDir, Format, LocalPkg (`*LocalPkgDef`) | HostNative | Always system. Compiled from a layer's per-format `localpkg:` map (the `charly` layer's `{pac: pkg/arch, rpm: pkg/fedora, deb: pkg/debian}`) at "step 2.5" (before tasks); `compileLocalPkgStep` resolves the target distro's format FIRST (`DistroDef.LocalPkgFormat`), then the layer's source for that format, so `Format` + `LocalPkg` come from the `format.<fmt>.local_pkg:` block in `build.yml` — EVERY command rendered from config, no hardcoded build/install/glob literals. On a localpkg-capable deploy target (`target: local` / `target: vm`) the HOST builds the format's source via `LocalPkg.BuildTemplate` (pac → `makepkg`; rpm/deb → a distro-matched podman container) into `LocalPkg.PkgGlob` artifacts, then installs them onto the target via `LocalPkg.InstallTemplate` — the format's AUTO-RESOLVING local-file install (`pacman -U` / `dnf install` / `apt-get install`), which pulls the package's repo deps automatically. There is NO dependency-closure builder (the aur-LAYER deploy path still reuses the shared `buildDepPkgsOnHost`/`transferAndInstallPkgs` leg, R3). `LocalPkg.Probe` gates the install leg; `LocalPkg.SourceSentinel` (`PKGBUILD`/`*.spec`/`debian/control`) marks the source dir. `resolveLocalPkgDir` walks up from `ProjectDir`, so a consumer nested under `image/<distro>` finds the superproject `pkg/<fmt>`. Skipped at image build (`OCITarget`) + on targets whose distro declares no localpkg-capable format (the layer's curl/COPY task is the fallback). Machinery: `ov/localpkg.go`; config: `build.yml format.<fmt>.local_pkg:`. |
| `RebootStep` | LayerName | HostNative | Always system; `Reverse()` empty. Emitted last when a layer sets `reboot: true`. Only `VmDeployTarget` acts on it (reboot guest + wait for return); OCI/pod/k8s skip; `LocalDeployTarget` skips + warns (never reboots the operator host). |

**`ShellSnippetStep` notes:**
- Compiled by `compileShellSnippetSteps` in `install_build.go` — applies the per-shell-wins-over-generic selection rule from `layer.Shell()`.
- OCITarget emits a `RUN mkdir -p ... && cat > <dest> <<EOF` heredoc with a sha256-derived end-marker (anti-collision).
- LocalDeployTarget / VmDeployTarget probe `command -v <shell>` once at the top of `Emit()`; absent shells become VenueSkip-style no-ops with a logged reason. UseDropin=true → whole-file write; UseDropin=false → `replaceOrAppendManagedBlock` against the existing rc file with a per-layer marker.
- Reverse: `ReverseOpRmFileSystem` / `ReverseOpRmFileUser` for drop-ins; `ReverseOpRemoveManaged` (with `Extra["marker"]=LayerName`) for managed-block append.
- Round-trip: `LabelShell` (`ai.opencharly.shell`) carries the merged set; `CollectShell` builds it at `charly box build` time, `ExtractMetadata` parses it at deploy time, `MergeDeployShell` overlays deploy.yml entries by id.

Each step's `Reverse()` emits typed `ReverseOp` values. Adding a step kind means: (a) define the struct in `install_plan.go`, (b) decide its Scope/Venue/Gate/Reverse, (c) add a case to each target's step dispatch (`emit*` in OCITarget; `exec*` in LocalDeployTarget), (d) ensure the compiler in `install_build.go` emits it.

## The compiler — `BuildDeployPlan`

```go
func BuildDeployPlan(layer *Layer, img *ResolvedImage, hostCtx HostContext) (*InstallPlan, error)
```

Pure — no I/O, no side effects. Given the same inputs, produces the same plan. Called:
- Once per layer during `charly box build` (OCITarget walks the combined output).
- Once per layer during a pod deploy (PodDeployTarget filters to `add_candy:` for overlay synthesis).
- Once per layer during a local deploy (LocalDeployTarget walks the combined output).

Pass `HostContext{Target: "host", Distro: ..., GlibcVersion: ...}` for host compilation; zero-value for build/container compilation.

Step emission order (mirrors today's `writeLayerSteps`):
1. `ShellHookStep` for `env:` + `path_append:` (deterministic map ordering).
2. ONE `SystemPackagesStep` for the image's primary format — resolved via the most-specific-first distro CASCADE over `ResolvedImage.Distro` (e.g. `[ubuntu:24.04, ubuntu]`) plus the layer's top-level `package:` base: packages UNION across every matching per-distro tag section, while `repo`/`copr`/`option`/`exclude`/`module` resolve most-specific-wins. No per-distro section ever shares a mutable format section, so a deb-family repo (trixie vs noble) resolves deterministically. The cascade lives in **ONE shared function `resolveCascadePackages` (`install_build.go`)** called by BOTH the deploy compiler (`compileSystemPackageSteps`) AND the image-build Containerfile emitter (`generate.go writeLayerSteps`) — there is exactly one package-resolution path, so a layer's packages are identical whether built into an image or applied at deploy. (Non-primary build formats like `aur` are a separate multi-stage builder concern, not a distro tag, and emit from their own format section.)
3. `TaskStep`(s) in YAML order.
4. `BuilderStep`(s) for each matching multi-stage or inline builder.
5. `ServicePackagedStep` / `ServiceCustomStep` from the `service:` list — per-entry routing via `IsPackaged()` + `ServiceSchema.SupportsPackaged`.

`MergePlans([]*InstallPlan, image, addLayers)` composes per-layer plans into a single whole-image plan for target-level walking (sudo batching, single dry-run output).

## The `DeployTarget` interface

```go
type DeployTarget interface {
    Name() string                 // "oci" | "pod" | "host" | "vm:<name>" | "k8s"
    Emit(plans []*InstallPlan, opts EmitOpts) error
}
```

Five implementations:

### `OCITarget` (`ov/build_target_oci.go`)
Emits Containerfile text. Consumes `phases.install.container` from build.yml (falls back to `install_template:`). For multi-stage builders, delegates to `Generator.buildStageContext` for the existing `BuildStageContext` template rendering. For tasks, delegates to `Generator.emitTasks` with a temporary layer-tasks swap so the existing per-verb emitters (`emitCopy`, `emitWrite`, `emitCmd`, `emitMkdirBatch`, ...) run unchanged.

Used by: `charly box build`, `charly box generate`, pod deploys with `add_candy:` (overlay Containerfile synthesis).

### `PodDeployTarget` (`ov/deploy_target_pod.go`)
Wraps the quadlet/podman pipeline with overlay-Containerfile synthesis for `add_candy:`. Picks an overlay tag deterministically from `(base-image, sorted-layer-set)`. Removed on `charly deploy del` unless `--keep-image`.

Used by: `charly deploy add <name>` (default `target: pod`) with `add_candy:` present.

### `LocalDeployTarget` (`ov/deploy_target_local.go`)
Walks the IR; groups contiguous same-`(Scope, Venue)` steps via `plan.StepsByVenue()`; emits one heredoc per batch. Full executor: writes service units (packaged + custom), env.d files, managed blocks, ledger entries. Invokes builder containers via `builder_run.go` for `VenueContainerBuilder` steps. Gates (`GateEnabled`) applied per step; skipped steps logged.

See `/charly-internals:local-infra` for supporting files (hostdistro, ledger, reverse_ops, shell_profile, builder_run, service_render, deploy_ref).

### `VmDeployTarget` (`ov/deploy_target_vm.go`)
Same IR walking as LocalDeployTarget, but shell bodies run via `ssh guest 'sudo bash -s'` through an `SSHExecutor` instead of local `sudo bash`. Ledger writes land on the **guest** filesystem; teardown runs in the guest via SSH. Preflight: `WaitForSSH` (120s) → `WaitForCloudInit` (cloud_image sources only) → `EnsureOvInGuest` (scp the `charly` binary per `VmOvInstall.Strategy`) → ensure guest ledger dir exists.

`DeployExecutor` is the abstraction that lets the same Emit logic retarget from local → SSH. `ShellExecutor` wraps local `bash -c` + file copy; `SSHExecutor` wraps ssh/scp via `golang.org/x/crypto/ssh` with persistent connection. Builder-container invocations (`VenueContainerBuilder` steps) run on the **host**, then artifacts scp into the guest — guests don't need podman installed.

Used by: `charly deploy add vm:<name> <ref>` / `charly deploy del vm:<name>`. See `/charly-internals:vm-deploy-target` for the full flow, `VmDeployState` persistence, and SSH-key idempotency.

### `K8sDeployTarget` (`ov/k8s_target.go`)
Emits a Kustomize `base/` + `overlays/` tree under `.opencharly/k8s/<name>/`. Does NOT execute anything; the generated manifests are applied via `kubectl apply -k` out-of-band. Cluster-specific choices (storage class, ingress class, cert issuer, secret backend) come from a **cluster profile** (`~/.config/charly/clusters/<name>.yaml`), NOT the InstallPlan — the plan describes what the workload needs; the profile describes how K8s provides it.

See `/charly-kubernetes:kubernetes` for the user-facing surface and profile layout.

## `StepBatch` batching

`InstallPlan.StepsByVenue()` partitions `Steps` into contiguous same-`(Scope, Venue)` runs. LocalDeployTarget uses this to emit one shell heredoc per batch:

| Batch | Emission form |
|---|---|
| `{ScopeSystem, VenueHostNative}` | `sudo bash <<'OV_ROOT' … OV_ROOT` |
| `{ScopeUser, VenueHostNative}` | `bash <<'OV_USER' … OV_USER` |
| `{_, VenueContainerBuilder}` | `podman run <builder> bash -s < script` |
| `{_, VenueSkip}` | Logged, no exec |

OCITarget doesn't batch — it emits each step in order as Containerfile directives; adjacent same-`USER` steps collapse naturally via the existing `USER` switching logic in `emitTasks`.

## Deferred home resolution (`{{.Home}}` + `InstallPlan.ResolveHome`)

Home-bearing step fields carry the deferred `HomeToken` (`{{.Home}}`) rather
than a home expanded at compile time. `compileShellHookStep` always emits the
token; `compileShellSnippetSteps` emits it for deploy targets (`hostCtx.Target`
== `host`/`vm`) and keeps `img.Home` for the container build. Each
`DeployTarget` calls `plan.ResolveHome(home)` once at emit with the home of the
**actual** destination:

| Target | Home used |
|---|---|
| `OCITarget` (build + pod-overlay) | `img.Home` (the image's runtime home) |
| `LocalDeployTarget` | host home (`ShellExecutor.ResolveHome`) |
| `VmDeployTarget` | GUEST home (`SSHExecutor.ResolveHome`) |

`ResolveHome` substitutes the token in `ShellHookStep` (EnvVars + PathAdd),
`ShellSnippetStep` (Snippet + Destination + PathAppend), and `FileStep.Dest`;
it deliberately skips `TaskStep` cmd/content bodies (`~`/`$HOME` there
shell-expand at runtime on the destination as the deploy user) and
`BuilderStep` (home resolved separately by `renderBuilderScript`). It is
idempotent. Baking `img.Home` at compile time was the VM `$HOME` bug: the
synthetic plan's Home was the host operator's, so a guest deploy wrote env.d
pointing at `/home/<operator>` and user-scope installs (npm -g, cargo) landed
in a root-owned path the guest user couldn't write. See
`/charly-internals:vm-deploy-target` "Guest-home resolution".

## `ReverseOp` catalogue

See `/charly-local:local-deploy` for the user-facing reverse-op table. The Go-level source of truth is `ReverseOpKind` in `install_plan.go`; each step's `Reverse()` method emits ops tagged with kind + targets + scope. Execution lives in `ov/reverse_ops.go` — one handler per kind, all routed through `runReverseOps(ops, executor)` in LIFO order.

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
- `install_build_test.go` — 8 integration tests that load real `candy/` via `ScanAllLayersWithConfig`; `testHostContextWithDistro` helper.
- `install_build.go:200` comment — canonical fixture docs.

When you add a step kind, add:
1. A scope/venue/gate/reverse unit test in `install_plan_test.go`.
2. An integration test in `install_build_test.go` exercising the compiler path.
3. Target-specific tests in `build_target_oci_test.go` and `deploy_target_local_test.go`.

## Cross-References

- `/charly-internals:local-infra` — supporting files (hostdistro, ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref)
- `/charly-internals:vm-deploy-target` — VmDeployTarget + DeployExecutor + SSHExecutor + VmDeployState
- `/charly-internals:vm-spec` — VmSpec shape that VmDeployTarget reads
- `/charly-internals:go` — overall Go code map; Kong CLI framework; mode-purity invariant
- `/charly-internals:generate-source` — Containerfile generation call graph; how OCITarget plugs into Generator
- `/charly-core:deploy` — user-facing `charly deploy add`/`del` surface (host / container / vm: / kubernetes)
- `/charly-local:local-deploy` — host-target user-facing behavior (ledger, gates, ReverseOps)
- `/charly-kubernetes:kubernetes` — K8s-target user-facing behavior (cluster profiles, Kustomize layout)
- `/charly-vm:vm` — VM command family; VmDeployTarget prerequisite (`charly vm create` before `charly deploy add vm:...`)
- `/charly-build:build` — build-mode user-facing surface; three-phase template story
- `/charly-image:layer` — candy.yml schema including unified `service:` that map to `ServicePackagedStep` / `ServiceCustomStep`

## When to Use This Skill

**MUST be invoked** when reading or modifying any of `ov/install_plan.go`, `ov/install_build.go`, `ov/build_target_*.go`, `ov/deploy_target_*.go`; when adding a new step kind, target, or reverse-op kind; or when debugging why a particular layer produces a plan that doesn't match expectations. Invoke BEFORE reading the source files or running Explore agents against this subsystem.
