---
name: install-plan
description: |
  The InstallPlan IR — the shared intermediate representation consumed by the DEPLOY
  targets: pod deploys (PodDeployTarget, incl. add_candy: overlay-Containerfile
  synthesis via OCITarget), VM deploys (VmDeployTarget over SSH), and external
  out-of-process deploys (externalDeployTarget over the executor reverse channel — the
  local/k8s/android substrates). The local substrate is EXTERNAL too (deploy:local,
  served out-of-process by candy/plugin-deploy-local): UNLIKE k8s it DOES consume the
  IR — the plugin walks the InstallPlan over the shared out-of-process walk
  (charly/plugin/kit.WalkPlans), driving the host for host-engine step kinds via the
  RunHostStep reverse-channel RPC. The k8s substrate is EXTERNAL (deploy:k8s, served
  out-of-process by candy/plugin-kube): it does NOT consume the IR — the host preresolver
  GENERATES a Kustomize tree (GenerateK8sKustomize, k8s_generate.go) and the plugin runs
  `kubectl apply -k`. Build-mode Containerfile emission is the SEPARATE writeCandySteps →
  emitTasks path.
  MUST be invoked before reading or modifying any of:
  charly/install_plan.go, charly/install_build.go, charly/build_target_oci.go,
  charly/deploy_target_pod.go, charly/deploy_target_vm.go,
  charly/deploy_target_external.go, charly/k8s_generate.go, charly/k8s_deploy_preresolve.go,
  charly/spec/deploy_wire.go, or when adding a new step kind / deploy target / reverse-op kind.
---

# InstallPlan IR — shared IR for build + deploy

## Overview

`charly` has several DEPLOY paths that all need to know "what does applying this layer mean?":

1. **Pod deploy** (`target: pod`, default) — `charly bundle add <name> <ref>` runs the image via quadlet, optionally building an overlay Containerfile (via `OCITarget`) when `add_candy:` is set.
2. **Local deploy** (`local:` substrate) — `charly bundle add <name> <ref>` applies the recipe to the destination machine's filesystem (`host: local` → direct shell; `host: <user@machine>` → SSH). Served OUT-OF-PROCESS by `candy/plugin-deploy-local` (`deploy:local`): `externalDeployTarget` hands the plugin this IR, the plugin walks it over the executor reverse channel (`charly/plugin/kit.WalkPlans`), and the host executor is `ShellExecutor` for `host:local`/absent or `SSHExecutor` for `host:<user@machine>` (picked by `rootExecutorForDeployNode`).
3. **VM deploy** (`target: vm`) — `charly bundle add vm:<name> <ref>` applies the recipe inside a running VM over SSH.
4. **Kubernetes deploy** (`target: k8s`) — `charly bundle add <name> --target k8s` emits a Kustomize base/overlays tree.
5. **External out-of-process deploy** — an external deploy-substrate plugin (`externalDeployTarget`) applies on the host venue over the executor reverse channel (`OpExecute`); see "External (out-of-process) deploy target" below.

All these DEPLOY paths are unified behind one IR. A pure compiler (`BuildDeployPlan`) turns `Layer + ResolvedBox + HostContext` into an `InstallPlan`; each deploy path is a `DeployTarget` consuming the plan.

**Build mode is a SEPARATE path, NOT the IR.** `charly box build` / `charly box generate` emit Containerfiles via the `writeCandySteps` → `emitTasks` generator (per-verb emitters in `charly/tasks.go`), walking each layer's ops directly. It shares the compiler helpers (`resolveCascadePackages`, `compileShellSnippetSteps`, `renderLocalPkgImageInstall` — R3) with the IR, but does NOT walk an `InstallPlan` or use `OCITarget`. `OCITarget` is itself deploy-mode: `PodDeployTarget` constructs it only for `add_candy:` overlay-Containerfile synthesis (`BuildDeployPlan` is called only by the deploy command path).

This skill is the single source of truth for the IR shape. Add a new step kind by editing `install_plan.go`, the compiler in `install_build.go`, and each target's `emit*` method — this skill lists every place that needs to stay in sync.

## File layout

| File | Role |
|---|---|
| `charly/install_plan.go` | IR types, enums, `InstallStep` interface, `ReverseOp`, `DeployTarget` interface, `EmitOpts`, `GateEnabled` |
| `charly/install_build.go` | `BuildDeployPlan` pure compiler: `Candy` → `InstallPlan` |
| `charly/build_target_oci.go` | `OCITarget` — emits Containerfile text from the IR; DEPLOY-mode (the `PodDeployTarget` `add_candy:` overlay synthesizer), NOT the `charly box build`/`generate` path |
| `charly/deploy_target_pod.go` | `PodDeployTarget` — synthesizes overlay Containerfile (via `OCITarget`) when `add_candy:` present, delegates to quadlet/start |
| `charly/deploy_chain.go` | `rootExecutorForDeployNode` — picks the root `DeployExecutor` for an external `local:` deploy node (`ShellExecutor` for `host:local`/absent, `SSHExecutor` for `host:<user@machine>`); the `deploy:local` substrate itself is served out-of-process by `candy/plugin-deploy-local`, which walks the plan via `charly/plugin/kit.WalkPlans` |
| `charly/deploy_target_vm.go` | `VmDeployTarget` — executes plan inside a running VM via SSH + scp |
| `charly/deploy_target_external.go` | `externalDeployTarget` — Add/Test/Update/Del for an OUT-OF-PROCESS deploy provider over the executor reverse channel (`OpExecute`); records teardown ops to the ledger keyed on `computeDeployID` |
| `charly/plugin_step_external.go` | `externalPluginStepProvider` — the `StepProvider` for `StepKindExternalPlugin` (a `run: plugin: <verb>` step served by an OUT-OF-PROCESS plugin): `EmitOCI`→`Invoke(OpEmit)` fragment, `EmitLocal`/`EmitVM`→`Invoke(OpExecute)` over the executor reverse channel; the `executorInvoker` discriminator + `executeExternalPluginStep` (records the reply's dynamic `ReverseOp`s to the `CandyRecord`) |
| `charly/spec/deploy_wire.go` | The deploy IR wire types shared with the plugin SDK: `Scope`, `ReverseOp` (+ `ReverseOpPluginScript`), `InstallPlanView`, `DeployVenue`, `DeployReply`, plus the build-time `BuildEnv` / `EmitReply` (verb/step `OpEmit`) / `BuilderResolveReply` (builder `OpResolve`: `{Stage, CopyArtifacts}`) |
| `charly/plugin_prescan.go` | The byte-gated, additive parse pre-scan that recognizes an EXTERNAL deploy SUBSTRATE word at config-parse time, before the provider connects |
| `charly/k8s_generate.go` | `GenerateK8sKustomize` — emits the Kustomize base/overlays tree from `(Capabilities, BundleNode, K8sSpec)`. Stays in core (egress-gated; second consumer `charly bundle from-box --target k8s`); called by the k8s deploy preresolver + from-box, NOT a `DeployTarget` |
| `charly/k8s_deploy_preresolve.go` | the HOST-side `deploy:k8s` preresolver (F1) — resolves the image Capabilities + the kind:k8s cluster template, runs `GenerateK8sKustomize`, and ships the overlay path in `DeployVenue.Substrate` (`spec.K8sDeployVenue`) for the out-of-process candy/plugin-kube `deploy:k8s` provider |
| `charly/deploy_executor*.go` | `DeployExecutor` interface + `ShellExecutor` + `SSHExecutor` — shell + file-copy abstraction shared by the external `local:` deploy path and `VmDeployTarget` |
| `charly/install_plan_test.go` | IR unit tests (scope/venue/gate/reverse derivations) |
| `charly/install_build_test.go` | Compiler integration tests (ripgrep, dev-tools, pixi layer) |

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
    CandiesIncluded []string     // ordered topo-sorted layer names (for merged plans)
    AddCandies      []string     // refs added via charly.yml add_candy: (for provenance)
    BuilderImage   string       // selected builder for VenueContainerBuilder steps
    Meta           map[string]string
}
```

One plan per layer when the compiler runs on a single `Candy`. For whole-image deploys, `MergePlan(plans, image, addCandies)` merges per-layer plans while preserving layer boundaries for refcount bookkeeping. `DeployID` is a deterministic 16-hex-char sha256 prefix over `(image, layer_order, add_layers)` — same inputs → same ID, so re-deploys are stable.

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

All thirteen concrete step kinds implement this interface. `Reverse()` is called at install time (not teardown) so the ledger records the exact reversal ops tied to the specific artifacts created. (`ExternalPluginStep` is the exception that proves the rule: its `Reverse()` is static-nil — its reversal ops are DYNAMIC, recorded from the plugin's `OpExecute` reply at emit time, not from the step itself.)

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
- `SystemPackages`, `Builder`, `Op`, `File`, `ServicePackaged`, `ServiceCustom`, `ShellHook`, `ShellSnippet`, `RepoChange`, `ApkInstall`, `LocalPkgInstall`, `Reboot`, `ExternalPlugin`.

The Go-internal vocabulary is the `allStepKinds` slice in `provider_step.go`; the `checkStepProviderBijection` gate asserts every kind has a registered `StepProvider` (add a kind → add it to BOTH).

The IR carries no image-fetch step kind. Deploys (any target) emit
zero image-pull / image-build steps; test-bed image preflight is a
separate, check-time concern handled by `charly/check_image_preflight.go`
(CLAUDE.md "Deploy fetches NOTHING speculative").

## The thirteen step kinds

| Kind | What it carries | Venue default | Scope derivation |
|---|---|---|---|
| `SystemPackagesStep` | Format (rpm/deb/pac), Phase, Packages, Repos, Options, Copr, Modules, Exclude, Keys, CacheMounts | HostNative | Always system |
| `BuilderStep` | Builder (pixi/npm/cargo/aur), BuilderImage, CandyDir, Phase, Artifacts, RawStageContext | ContainerBuilder | aur→system, others→user |
| `OpStep` | Op (raw), CandyName, CandyDir, CtxPath, ResolvedUser | HostNative | From ResolvedUser (root or 0 → system; else user) |
| `FileStep` | Source, Dest, Mode, Owner, CandyName | HostNative | `pathIsSystemScoped(Dest)` |
| `ServicePackagedStep` | Unit, TargetScope, Enable, OverridesText, OverridesPath, CandyName, PriorEnabled | HostNative | TargetScope field |
| `ServiceCustomStep` | Name, UnitText, UnitPath, TargetScope, Enable, CandyName | HostNative | TargetScope field |
| `ShellHookStep` | CandyName, EnvVars, PathAdd, EnvFile | HostNative | Always user-profile |
| `ShellSnippetStep` | CandyName, Origin, Shell (bash/zsh/fish/sh), Snippet, PathAppend, Destination, Marker, UseDropin, Priority | HostNative | `pathIsSystemScoped(Destination)` (system for container drop-ins, user-profile for ~/.bashrc etc.) |
| `RepoChangeStep` | Format, File, Content, Checksum, CandyName | HostNative | Always system |
| `ApkInstallStep` | Packages (apk specs), CandyName, CandyDir | HostNative | Always system. Only `target: android` executes it; every other target records a skip. |
| `LocalPkgInstallStep` | PkgbuildRef, CandyName, CandyDir, ProjectDir, Format, LocalPkg (`*LocalPkgDef`) | HostNative | Always system. Compiled from a layer's per-format `localpkg:` map (the `charly` layer's `{pac: pkg/arch, rpm: pkg/fedora, deb: pkg/debian}`) at "step 2.5" (before tasks); `compileLocalPkgStep` resolves the target distro's format FIRST (`DistroDef.LocalPkgFormat`), then the layer's source for that format, so `Format` + `LocalPkg` come from the `format.<fmt>.local_pkg:` block in the embedded `charly/charly.yml` — EVERY command rendered from config, no hardcoded build/install/glob literals. On a localpkg-capable deploy target (`target: local` / `target: vm`) the HOST builds the format's source via `LocalPkg.BuildTemplate` (pac → `makepkg`; rpm/deb → a distro-matched podman container) into `LocalPkg.PkgGlob` artifacts, then installs them onto the target via `LocalPkg.InstallTemplate` — the format's AUTO-RESOLVING local-file install (`pacman -U` / `dnf install` / `apt-get install`), which pulls the package's repo deps automatically. There is NO dependency-closure builder (the aur-LAYER deploy path still reuses the shared `buildDepPkgsOnHost`/`transferAndInstallPkgs` leg, R3). `LocalPkg.Probe` gates the install leg; `LocalPkg.SourceSentinel` (`PKGBUILD`/`*.spec`/`debian/control`) marks the source dir. `resolveLocalPkgDir` walks up from `ProjectDir`, so a consumer nested under `box/<distro>` finds the superproject `pkg/<fmt>`. At IMAGE build the install is **mode-switched by box type** (unified in `renderLocalPkgImageInstall`, shared by `OCITarget` + `generate.go writeCandySteps` — R3): a PRODUCTION box DOWNLOADS the published release (`LocalPkg.DownloadTemplate`), while a DISPOSABLE check bed BUILDS the package from LOCAL in-development source and COPYs it in — see "Check-vs-production charly toolchain" below. Skipped only on a distro with no localpkg-capable format (the layer's task fallback). Machinery: `charly/localpkg.go`; config: the embedded `charly/charly.yml` `format.<fmt>.local_pkg:` block. |
| `RebootStep` | CandyName | HostNative | Always system; `Reverse()` empty. Emitted last when a layer sets `reboot: true`. Only `VmDeployTarget` acts on it (reboot guest + wait for return); OCI/pod/k8s skip; the external `local:` deploy skips + warns (never reboots the operator host). |
| `ExternalPluginStep` | Op (the `run: plugin:` op — verb word + `plugin_input` + `RunAs`), CandyName, ResolvedUser, Distros | HostNative | From ResolvedUser (`opStepScope`: root/empty → system; else user). `Reverse()` static-nil — teardown ops are recorded DYNAMICALLY from the plugin's `OpExecute` reply. A `run: plugin: <verb>` step whose verb is served by an EXTERNAL (out-of-process) plugin — see "ExternalPluginStep notes" below. |

### Check-vs-production charly toolchain (the `--dev-local-pkg` distinction)

A `localpkg:` candy (the `charly` toolchain) installs the charly binary as a
proper OS package on every distro image. The BINARY SOURCE depends on the box
type — a hard, GENERIC distinction, NEVER mixed, decided in ONE place
(`renderLocalPkgImageInstall`, `charly/localpkg.go`):

| Box type | charly binary source | How |
|---|---|---|
| **Disposable check box** (a `disposable: true` deploy) | latest **in-development** | the check-bed runner ALWAYS passes `charly box build --dev-local-pkg`, so the localpkg is BUILT from local source (`pkg/<fmt>` + `charly/`, via `buildLocalPkgOnHost`) and COPY'd into the image — a bed tests the charly code under development |
| **Production box** (any normal `charly box build`) | latest **published** | the default: the localpkg DOWNLOADS the published release (`releases/latest/download/opencharly-<arch>.<fmt>`) |

Both install via the SAME dep-resolving `InstallTemplate` (`pacman -U` / `dnf
install` / `apt-get install`) — only the package SOURCE differs, so the toolchain
is OS-tracked either way. The switch flows through `Generator.DevLocalPkg` ←
`BuildCmd.DevLocalPkg` (`--dev-local-pkg`); the check-bed runner sets it for EVERY
bed image build (`check_bed_run.go`), a production build never does. Generic
across all kinds + all localpkg candies. A dev build that cannot locate its local
source **HARD-ERRORS** — it NEVER silently falls back to the release (R4), so a
bed can never accidentally test a stale published binary. Net: a disposable check
bed always exercises the in-development charly; a real box ships the released one.

**`ShellSnippetStep` notes:**
- Compiled by `compileShellSnippetSteps` in `install_build.go` — applies the per-shell-wins-over-generic selection rule from `layer.Shell()`.
- OCITarget emits a `RUN mkdir -p ... && cat > <dest> <<EOF` heredoc with a sha256-derived end-marker (anti-collision).
- The `local:` (external plugin) and `vm:` (`VmDeployTarget`) deploy walks probe `command -v <shell>` once at the top of the walk; absent shells become VenueSkip-style no-ops with a logged reason. UseDropin=true → whole-file write; UseDropin=false → `replaceOrAppendManagedBlock` against the existing rc file with a per-layer marker.
- Reverse: `ReverseOpRmFileSystem` / `ReverseOpRmFileUser` for drop-ins; `ReverseOpRemoveManaged` (with `Extra["marker"]=CandyName`) for managed-block append.
- Round-trip: `LabelShell` (`ai.opencharly.shell`) carries the merged set; `CollectShell` builds it at `charly box build` time, `ExtractMetadata` parses it at deploy time, `MergeDeployShell` overlays charly.yml entries by id.

**`ExternalPluginStep` notes** (`charly/plugin_step_external.go`, `externalPluginStepProvider`):

- **What it is.** The install-timeline IR node for a `run: plugin: <verb>` step whose verb is served by an EXTERNAL (out-of-process) plugin — a `grpcProvider`, not a built-in `TypedStepProvider` (package/service) nor a `ProvisionActor` (the state-provision shell verbs) nor the `command` install verb. It is operator-authorized build/deploy-time plugin execution: the candy author opted in by composing the verb-step.
- **Routing (`compileActOp`, `install_build.go`).** A `plugin:` verb resolves through `providerRegistry.ResolveVerb`; if its provider is NOT a `TypedStepProvider` but DOES satisfy `executorInvoker` (an `InvokeWithExecutor` method, implemented SOLELY by the out-of-process `grpcProvider`), it routes to `ExternalPluginStep`. Every builtin `ProvisionActor` verb and `command` fall through to the generic `OpStep` path, unchanged. The `executorInvoker` capability is the precise discriminator — it mirrors the build-context `BuildEmitter` marker (`provider_verb.go`), placement-agnostic above the registry.
- **EmitOCI — the BUILD venue** (image build / pod-overlay Containerfile): `Invoke(OpEmit)` via the SHARED `emitPluginFragment` seam (R3 — the SAME path `tasks.go`'s `emitTasks` `case "plugin"` takes for an external verb), splicing the plugin's Containerfile FRAGMENT verbatim. It cannot deploy-execute at build (no live venue); a deploy-only plugin (empty `OpEmit` fragment) fails loudly at `emitPluginFragment`'s empty-fragment guard, never bakes nothing silently.
- **EmitLocal / EmitVM — the DEPLOY venue** (the install runs ON the target, not into an image): `Invoke(OpExecute)` WITH the live `DeployExecutor` on the go-plugin broker (the E3b reverse channel), so the plugin runs its deploy-context effect (RunSystem/RunUser) on the real venue it cannot hold across the process boundary, and RETURNS its teardown `ReverseOp`s. The target appends `reply.ReverseOps` to the `CandyRecord` (`EmitLocal` → the host ledger; `EmitVM` → the guest-side ledger) — record-and-replay: only recorded ops are reversed at `charly bundle del`, reusing the SAME `spec.DeployReply` / `ReverseOp` wire as the external deploy TARGET (`deploy_target_external.go`, R3). `plugin_input` rides `op.Params` UNWRAPPED; a `spec.DeployVenue` rides `op.Env`. A `DryRun` short-circuits BEFORE the wire call (Invoke IS the apply). The shared `executeExternalPluginStep` backs both venues so they cannot drift.
- **`OpExecute` is the deploy-context counterpart of `OpEmit` (build).** Same verb-step, two legs: bake a Containerfile fragment at build (`OpEmit`), or execute the effect on a live `local:`/`vm:` target at deploy (`OpExecute`) — picked by venue, placement-agnostic.
- **Self-registers** via `registerDedicatedBuiltin(externalPluginStepProvider{})`, like every other dedicated step provider; the `StepProvider` it implements lives in `provider_step.go`.

Each step's `Reverse()` emits typed `ReverseOp` values. Adding a step kind means: (a) define the struct in `install_plan.go`, (b) decide its Scope/Venue/Gate/Reverse, (c) add a case to each target's step dispatch (`emit*` in OCITarget; the shared `charly/plugin/kit.WalkPlans` for the plugin-renderable legs, plus `RunHostStep` if it is a host-engine kind), (d) ensure the compiler in `install_build.go` emits it.

## The compiler — `BuildDeployPlan`

```go
func BuildDeployPlan(layer *Candy, img *ResolvedBox, hostCtx HostContext) (*InstallPlan, error)
```

Pure — no I/O, no side effects. Given the same inputs, produces the same plan. Called ONLY by the deploy command path (`bundle_add_cmd.go`), never by `charly box build`/`generate`:
- Once per layer during a pod deploy with `add_candy:` (PodDeployTarget filters to `add_candy:`, OCITarget walks the combined output for overlay synthesis).
- Once per layer during a VM or external deploy (local/k8s/android) — the target walks the combined output.

Pass `HostContext{Target: "host", Distro: ..., GlibcVersion: ...}` for host compilation; zero-value for the pod-overlay container compilation.

Step emission order (mirrors today's `writeCandySteps`):
1. `ShellHookStep` for `env:` + `path_append:` (deterministic map ordering).
2. ONE `SystemPackagesStep` for the image's primary format — resolved via the most-specific-first distro CASCADE over `ResolvedBox.Distro` (e.g. `[ubuntu:24.04, ubuntu]`) plus the layer's top-level `package:` base: packages UNION across every matching per-distro tag section, while `repo`/`copr`/`option`/`exclude`/`module` resolve most-specific-wins. No per-distro section ever shares a mutable format section, so a deb-family repo (trixie vs noble) resolves deterministically. The cascade lives in **ONE shared function `resolveCascadePackages` (`install_build.go`)** called by BOTH the deploy compiler (`compileSystemPackageSteps`) AND the image-build Containerfile emitter (`generate.go writeCandySteps`) — there is exactly one package-resolution path, so a layer's packages are identical whether built into an image or applied at deploy. (Non-primary build formats like `aur` are a separate multi-stage builder concern, not a distro tag, and emit from their own format section.)
3. `OpStep`(s) in YAML order.
4. `BuilderStep`(s) for each matching multi-stage or inline builder.
5. `ServicePackagedStep` / `ServiceCustomStep` from the `service:` list — per-entry routing via `IsPackaged()` + `ServiceSchema.SupportsPackaged`.

`MergePlan([]*InstallPlan, image, addCandies)` composes per-layer plans into a single whole-image plan for target-level walking (sudo batching, single dry-run output).

## The `DeployTarget` interface

```go
type DeployTarget interface {
    Name() string                 // "oci" | "pod" | "host" | "vm:<name>"
    Emit(plans []*InstallPlan, opts EmitOpts) error
}
```

Four implementations (`OCITarget`, `PodDeployTarget`, `VmDeployTarget`, `externalDeployTarget`):

### `OCITarget` (`charly/build_target_oci.go`)
Emits Containerfile text. Consumes `phases.install.container` from the embedded `charly/charly.yml` build vocabulary (falls back to `install_template:`). For multi-stage builders, delegates to `Generator.buildStageContext` for the existing `BuildStageContext` template rendering. For tasks, delegates to `Generator.emitTasks` with a temporary layer-tasks swap so the existing per-verb emitters (`emitCopy`, `emitWrite`, `emitCmd`, `emitMkdirBatch`, ...) run unchanged.

Used by: pod deploys with `add_candy:` (overlay Containerfile synthesis) — the ONLY `OCITarget` construction site. `charly box build` / `charly box generate` do NOT use `OCITarget`; build-mode Containerfile emission is the separate `writeCandySteps` → `emitTasks` generator (see the Overview "Build mode is a SEPARATE path").

### `PodDeployTarget` (`charly/deploy_target_pod.go`)
Wraps the quadlet/podman pipeline with overlay-Containerfile synthesis for `add_candy:`. Picks an overlay tag deterministically from `(base-image, sorted-layer-set)`. Removed on `charly bundle del` unless `--keep-image`.

Used by: `charly bundle add <name>` (default `target: pod`) with `add_candy:` present.

### `local:` is EXTERNAL (`deploy:local`, candy/plugin-deploy-local) — consumes the IR over the reverse channel
There is no in-proc local DeployTarget. `local:` resolves to `externalDeployTarget` (below), served out-of-process by `candy/plugin-deploy-local`'s `deploy:local` provider (F1). UNLIKE k8s, the local substrate DOES consume the InstallPlan IR: the plugin walks the plan via the shared out-of-process walk (`charly/plugin/kit.WalkPlans`), groups contiguous same-`(Scope, Venue)` steps via `StepsByVenue()`, emits one heredoc per batch, and writes service units (packaged + custom), env.d files, and managed blocks (the host records the returned teardown ops to the ledger via `install_ledger.go`). The plugin owns the walk ORDERING but cannot execute the host-engine step kinds itself — `BuilderStep`, `LocalPkgInstallStep`, `SystemPackagesStep` (host renders via `DistroConfig`), act-verb `OpStep` (host registry `resolveProvisionScript`), and `ExternalPluginStep` (nested plugin dispatch) are driven on the HOST via the `RunHostStep` reverse-channel RPC (the host owns the engine/registry). The host executor is `ShellExecutor` for `host:local`/absent and `SSHExecutor` for `host:<user@machine>`, picked by `rootExecutorForDeployNode` (`charly/deploy_chain.go`). See `/charly-local:local-deploy` for the user-facing surface and `/charly-internals:local-infra` for supporting files (hostdistro, ledger, reverse_ops, shell_profile, builder_run, service_render, deploy_ref).

### `VmDeployTarget` (`charly/deploy_target_vm.go`)
Same IR walking as a local deploy, but shell bodies run via `ssh guest 'sudo bash -s'` through an `SSHExecutor` instead of local `sudo bash`. Ledger writes land on the **guest** filesystem; teardown runs in the guest via SSH. Preflight: `WaitForSSH` (120s) → `WaitForCloudInit` (cloud_image sources only) → `EnsureCharlyInGuest` (scp the `charly` binary per `VmCharlyInstall.Strategy`) → ensure guest ledger dir exists.

`DeployExecutor` is the abstraction that lets the same Emit logic retarget from local → SSH. `ShellExecutor` wraps local `bash -c` + file copy; `SSHExecutor` wraps ssh/scp via `golang.org/x/crypto/ssh` with persistent connection. Builder-container invocations (`VenueContainerBuilder` steps) run on the **host**, then artifacts scp into the guest — guests don't need podman installed.

Used by: `charly bundle add vm:<name> <ref>` / `charly bundle del vm:<name>`. See `/charly-internals:vm-deploy-target` for the full flow, `VmDeployState` persistence, and SSH-key idempotency.

### k8s is EXTERNAL (`deploy:k8s`, candy/plugin-kube) — NOT an IR-consuming DeployTarget
There is no in-proc k8s DeployTarget. `target: k8s` resolves to `externalDeployTarget` (below), served out-of-process by candy/plugin-kube's `deploy:k8s` provider (F1, beside its `kube:` verb). The Kustomize generation cannot leave core — `GenerateK8sKustomize` (`charly/k8s_generate.go`) consumes the package-main `Capabilities`/`BoxMetadata` type + the CUE egress gate (`#K8sObject` / `#Kustomization`) the plugin cannot reach, AND it has a second in-core consumer (`charly bundle from-box --target k8s`). So the host-side `deploy:k8s` preresolver (`charly/k8s_deploy_preresolve.go`) resolves the image Capabilities + the kind:k8s cluster template, RUNS `GenerateK8sKustomize` (the egress-validated `base/` + `overlays/` tree under `.opencharly/k8s/<name>/`), and ships the overlay path in `DeployVenue.Substrate` (`spec.K8sDeployVenue`). The plugin then runs `kubectl --context <ctx> apply -k` (the LIVE cluster I/O it owns) and returns the `kubectl delete -k` + tree-removal teardown op the host records. k8s does NOT consume the InstallPlan IR (a k8s deploy compiles no primary image plan). Cluster-specific choices come from a `kind: k8s` cluster template (the `k8s:` entity), not the InstallPlan. See `/charly-kubernetes:kubernetes` + `/charly-internals:plugin`.

### `externalDeployTarget` (`charly/deploy_target_external.go`)
The `UnifiedDeployTarget` adapter for an OUT-OF-PROCESS deploy provider (a `grpcProvider` whose class is `deploy`). The full lifecycle rides over the host-served executor reverse channel — placement-invisible above the registry, so an external deploy substrate is a free authoring choice:

- **Add** marshals the deployment's `InstallPlan`s (as `spec.InstallPlanView`, the JSON-roundtrippable provenance view) into `op.Params` and a `spec.DeployVenue` descriptor into `op.Env`, then `InvokeWithExecutor(OpExecute)` with the host's executor on the go-plugin broker so the plugin runs the deployment's shell/SSH ops on the real venue it cannot hold across the process boundary. It decodes the structured `spec.DeployReply` (`{reverse_ops, record}`) and writes its `ReverseOps` + provenance into the ledger via the SAME `install_ledger.go` path a built-in Add uses.
- **Test** runs the deploy-scope checks HOST-SIDE — `runUnifiedTargetChecks` against the executor (the plugin is not involved; the checks are in-proc `CheckVerbProvider`s, R3).
- **Update** re-`Invoke`s `OpExecute` with fresh plans — an idempotent re-Add (the candy's ledger `ReverseOps` are REPLACED, not appended).
- **Del** replays the RECORDED `ReverseOps` from the ledger (no plugin call) via the shared `teardownHostDeploy` — the record-and-replay invariant: only recorded ops are reversed, never recomputed.

The ledger key is `computeDeployID(name, nil, nil)` — derived from the deploy name alone (so the host-venue `Kind()=="host"` never collides with another host-venue deploy on the ledger scan). A bed/deploy that uses an external deploy SUBSTRATE word is recognized at config-PARSE time by the byte-gated, additive declaration pre-scan (`plugin_prescan.go`) before the provider connects; `charly check live` / `charly check <verb>` route an external deploy host-side via the shared `checkLocalTarget` classifier (`check_venue.go`) — the host-venue path the externalized `local:` substrate itself takes (R3). The wire types (`InstallPlanView`, `DeployVenue`, `DeployReply`, `ReverseOp`, `ReverseOpPluginScript`) live in `charly/spec/deploy_wire.go`, SDK-importable so an out-of-tree deploy plugin constructs the same structs (R3).

## Op selectors at build vs deploy

A provider's `Invoke` carries an `op.Op` selector (`charly/provider.go`). Three drive THIS subsystem (the build/deploy IR), placement-agnostically (in-proc for a builtin, go-plugin gRPC for an external); a fourth, `OpRun`, is the runtime/CLI selector OUTSIDE the install-plan IR (listed here for selector completeness):

- **`OpEmit`** is INVOKED at IMAGE-BUILD time — a `run:` plugin verb / plugin step returns a `spec.EmitReply.Fragment` (with a `spec.BuildEnv` descriptor in `op.Env`) that the generator splices verbatim into the Containerfile (egress-validated). This is a BUILD-mode concern, dispatched from `emitTasks` (`charly/tasks.go`), NOT the IR/`OCITarget` — EXCEPT the pod-overlay `OCITarget`, where `ExternalPluginStep.EmitOCI` reuses the SAME `emitPluginFragment` seam (R3). See `/charly-build:generate` + `/charly-internals:generate-source`.
- **`OpResolve`** is ALSO INVOKED at IMAGE-BUILD time — the BUILDER leg. An EXTERNAL `ClassBuilder` provider selected by a candy's `external_builder: <word>` field returns a `spec.BuilderResolveReply` (`{Stage, CopyArtifacts}`, with a `spec.BuildEnv` descriptor in `op.Env` + the candy name in `op.Params`): `Stage` (a `FROM <ref> AS <name>` block) splices PRE-main-FROM, `CopyArtifacts` (`COPY --from=<stage> …`) POST-main-FROM. Dispatched from `emitExternalBuilderStages` / `emitExternalBuilderArtifacts` (`charly/generate.go`), the multi-stage counterpart of the verb/step `OpEmit` leg. See `/charly-build:generate` + `/charly-internals:generate-source`.
- **`OpExecute`** drives EXTERNAL deploy-context execution over the executor reverse channel — `externalDeployTarget`'s Add/Update `InvokeWithExecutor(OpExecute)` (the deploy-substrate lifecycle), AND `ExternalPluginStep.EmitLocal`/`EmitVM` for a `run: plugin: <verb>` step on a `local:`/`vm:` target — both `InvokeWithExecutor(OpExecute)` with the host's live executor, both decoding the `spec.DeployReply` teardown record (above). `OpExecute` is the deploy-context counterpart of the build-context `OpEmit`: the SAME external verb-step bakes a fragment at build (`OpEmit`) and executes its effect on a live target at deploy (`OpExecute`), picked by venue.
- **`OpRun`** is the RUNTIME / CLI selector, OUTSIDE the install-plan IR (it neither builds an image nor applies a deploy plan): it runs a check verb / live-container probe (`provider_checkenv.go` `invokeVerbProvider`), AND it dispatches an EXTERNAL COMMAND plugin's `charly <word>` subcommand — `provider_command_external.go` `dispatchExternalCommand` forwards the pass-through CLI tokens as `op.Params = {"args":[…]}` (marshalled directly, no `plugin_input` envelope) on an `Invoke(OpRun)` to the lazy-connected out-of-process command provider. Owned by `/charly-internals:plugin` (the command class) + `/charly-check:check` (the verb probes).

## `StepBatch` batching

`InstallPlan.StepsByVenue()` partitions `Steps` into contiguous same-`(Scope, Venue)` runs. The `local:`/`vm:` deploy walk uses this to emit one shell heredoc per batch:

| Batch | Emission form |
|---|---|
| `{ScopeSystem, VenueHostNative}` | `sudo bash <<'CHARLY_ROOT' … CHARLY_ROOT` |
| `{ScopeUser, VenueHostNative}` | `bash <<'CHARLY_USER' … CHARLY_USER` |
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
| `OCITarget` (pod-overlay image build) | `img.Home` (the image's runtime home) |
| external `local:` deploy | host home (`ShellExecutor.ResolveHome`) |
| `VmDeployTarget` | GUEST home (`SSHExecutor.ResolveHome`) |

`ResolveHome` substitutes the token in `ShellHookStep` (EnvVars + PathAdd),
`ShellSnippetStep` (Snippet + Destination + PathAppend), and `FileStep.Dest`;
it deliberately skips `OpStep` command/content bodies (`~`/`$HOME` there
shell-expand at runtime on the destination as the deploy user) and
`BuilderStep` (home resolved separately by `renderBuilderScript`). It is
idempotent. Baking `img.Home` at compile time was the VM `$HOME` bug: the
synthetic plan's Home was the host operator's, so a guest deploy wrote env.d
pointing at `/home/<operator>` and user-scope installs (npm -g, cargo) landed
in a root-owned path the guest user couldn't write. See
`/charly-internals:vm-deploy-target` "Guest-home resolution".

## `ReverseOp` catalogue

See `/charly-local:local-deploy` for the user-facing reverse-op table. The Go-level source of truth is `ReverseOpKind` in `install_plan.go`; each step's `Reverse()` method emits ops tagged with kind + targets + scope. Execution lives in `charly/reverse_ops.go` — one handler per kind, all routed through `runReverseOps(ops, executor)` in LIFO order.

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

CLI flags on `BundleAddCmd` / `BundleDelCmd` populate this struct; each target reads what it needs. `AssumeYes` enables all three opt-in gates (via `GateEnabled`).

## Testing

- `install_plan_test.go` — 13 unit tests over step-kind derivations (scope/venue/gate/reverse).
- `install_build_test.go` — 8 integration tests that load real `candy/` via `ScanAllCandyWithConfig`; `testHostContextWithDistro` helper.
- `install_build.go:200` comment — canonical fixture docs.

When you add a step kind, add:
1. A scope/venue/gate/reverse unit test in `install_plan_test.go`.
2. An integration test in `install_build_test.go` exercising the compiler path.
3. Target-specific tests in `build_target_oci_test.go`, the shared out-of-process walk in `charly/plugin/kit/walk_test.go`, and the host-engine channel in `plugin_executor_hoststep_test.go`.

## Cross-References

- `/charly-internals:local-infra` — supporting files (hostdistro, ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref)
- `/charly-internals:vm-deploy-target` — VmDeployTarget + DeployExecutor + SSHExecutor + VmDeployState
- `/charly-internals:vm-spec` — VmSpec shape that VmDeployTarget reads
- `/charly-internals:go` — overall Go code map; Kong CLI framework; mode-purity invariant
- `/charly-internals:generate-source` — Containerfile generation call graph; how OCITarget plugs into Generator
- `/charly-core:deploy` — user-facing `charly bundle add`/`del` surface (host / container / vm: / kubernetes)
- `/charly-local:local-deploy` — host-target user-facing behavior (ledger, gates, ReverseOps)
- `/charly-kubernetes:kubernetes` — K8s-target user-facing behavior (cluster profiles, Kustomize layout)
- `/charly-vm:vm` — VM command family; VmDeployTarget prerequisite (`charly vm create` before `charly bundle add vm:...`)
- `/charly-build:build` — build-mode user-facing surface; three-phase template story
- `/charly-image:layer` — charly.yml schema including unified `service:` that map to `ServicePackagedStep` / `ServiceCustomStep`

## When to Use This Skill

**MUST be invoked** when reading or modifying any of `charly/install_plan.go`, `charly/install_build.go`, `charly/build_target_*.go`, `charly/deploy_target_*.go`; when adding a new step kind, target, or reverse-op kind; or when debugging why a particular layer produces a plan that doesn't match expectations. Invoke BEFORE reading the source files or running Explore agents against this subsystem.
