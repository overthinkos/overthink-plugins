---
name: go
description: |
  Go CLI development: building the charly binary, running tests, understanding
  the source code structure.
  MUST be invoked before reading or modifying any Go source file in charly/.
---

# Go - CLI Development

## Overview

The `charly` CLI is a Go program in the `charly/` directory. It uses the Kong CLI framework, go-containerregistry for OCI operations, and YAML parsing for configuration. All computation, validation, and building logic lives in Go. Taskfiles are used only for bootstrapping (building charly itself).

### Unified YAML loader (`LoadUnified`)

The unified format's entry point is `LoadUnified(dir)` at `charly/unified.go`. It reads `<dir>/charly.yml`, recursively resolves the `import:` statement (max depth 8, cycle-safe via visited set), parses every file as a **YAML multi-document stream** (so bundle files with `---` separators work), and routes each document by shape:

- **Root-shape doc** — has any of `version:`, `import:`, `discover:`, `defaults:`, `distro:`, `builder:`, `init:`, `box:`, `candy:`, `deploy:` → parsed as `UnifiedFile`, merged root-wins.
- **Kind-keyed doc** — has exactly one of `candy:`, `box:`, `deploy:`, `builder:`, `distro:`, `init:` (and the other kinds) + a `name:` inside → registered under `name:` in the matching map.
- **Ambiguous** (both root and kind keys) → loader error with file + doc-index.

`UnifiedFile.ApplyDiscover(rootDir)` walks the **flat generic `discover:` list** after initial merge. `discover:` is `DiscoverConfig` = `[]ScanSpec` (`{path, recursive, manifest}`) — no kind dimension. For each spec, `findEntityDirs` finds directories containing the spec's `manifest` (default `UnifiedFileName` — `charly.yml`, the ONE filename the code knows; the former separate `DefaultManifest` constant is deleted; a missing discover path is a no-op, not an error), and `applyDiscoveredManifest` routes each discovered document BY SHAPE via `firstKindKey`: a `candy:`-shaped doc registers a lazy `From:` directory reference (`scanCandy` parses + validates it later), every other shape decodes + merges via `mergeKindDoc`. Explicit map entries always win over discovered entries. **`ApplyDiscover` runs in the loader's main path (`loadUnifiedInto` depth-0 boundary, for the root AND every namespace), so discovered `box:` docs reach `ProjectConfig` — not just the layer-loading path.** There is no per-kind discovery function and no `entityKind.Filename` — both were deleted; the kind vocabulary lives only in `kindKeys`/`kindKeysSet` for shape classification.

Projections to today's concrete types: `ProjectConfig()` → `*Config`, `ProjectDistroConfig()` → `*DistroConfig`, etc. Existing `LoadConfig` / `LoadBuildConfigForBox` / `LoadDeployConfig` continue to work unchanged — migration to the unified entry point is incremental.

**CUE-source front-end (`charly/cue_source.go`).** The loader has ONE document-interpretation path (`mergeUnifiedDocs`), fed by two source FRONT-ENDS: the yaml.v3 decoder for on-disk `charly.yml`, and `compileCUEToYAML` for CUE source. The latter compiles a CUE document with `cueSchemaCtx`, requires it concrete (`cue.Concrete(true)` — a config is data, not a schema; CUE definitions + hidden `_` helpers used to DRY the source resolve at compile time and never export), and re-encodes it to YAML bytes that flow through the UNCHANGED `mergeUnifiedDocs` — so a CUE-authored config is parsed EXACTLY like a `charly.yml` after the front-end (no parallel routing/normalize/merge — R3). Used by the binary-embedded default config, now authored as `charly/charly.cue` (`embed_defaults.go`); project on-disk config stays YAML-only. The embedded vocabulary is schema-validated against `charly/schema` (`#Distro`/`#Builder`/`#Init`/`#Resource`/`#Sidecar`) through the shared `validateVocabularyCollections` helper (`validate.go`, also used by `charly box validate` for project files) — guarded by `TestEmbeddedDefaults_SchemaConformance` and the byte-for-byte `TestEmbeddedCUE_DataEquivalentToLegacyYAML`.

### import-namespace loader (`UnifiedFile.Import` + `Namespaces`)

`UnifiedFile.Import` (type `ImportList`, YAML tag `import`) is the single composition statement. Its custom `UnmarshalYAML` accepts a mixed-shape sequence: a scalar item → `ImportEntry{Ref: …}` (flat, `Namespace == ""`); a single-key mapping item → `ImportEntry{Namespace: alias, Ref: …}` (namespaced). A matching `MarshalYAML` round-trips both shapes (migrators rely on it). `validateNamespaceAlias` enforces a bare lowercase-hyphenated alias (no dots).

`loadUnifiedInto` processes the queue:

- **Flat entries** are loaded and root-merged into the importing `UnifiedFile` (same root-wins merge that drives same-repo file splits + shared `build.yml`).
- **Namespaced entries** call `loadNamespaceCached(ref, base, nsCache, loadingRepos)`, which loads the target as a fully-resolved, **isolated** `UnifiedFile` (its own flat imports + its own namespaced imports, with a FRESH `visited` set for its file-cycle detection) and mounts it under `merged.Namespaces[alias]`. These entries are NOT flat-merged into the root maps — they are referenced qualified.

`UnifiedFile.Namespaces` (`map[string]*UnifiedFile`, YAML tag `-`, never authored directly) holds the mounted children; `projectConfigCached` projects it to `Config.Namespaces` (`map[string]*Config`, pointer-keyed cache → self-references project safely).

**Cycle-break by REPO IDENTITY (`ns_identity.go`), not pinned version.** Two maps cooperate in `loadNamespaceCached`: `nsCache` is the version-keyed (`canonicalRef`: `repo@version/subpath`) *diamond memo* — it dedups identical refs across a load; `loadingRepos` is the *ancestor/cycle* set, keyed by REPO IDENTITY (`nsRepoIdentity`: a remote ref's `RepoPath`, or a local path's `git remote origin`). BEFORE any fetch, if the ref's repo identity is already in `loadingRepos` (an ancestor still on the load stack), the loader returns that in-progress node — so an import cycle between two projects that import each other (or a transitive back-import of an ancestor still being loaded) terminates even when **the loop's pins diverge**: a back-reference to a DIFFERENT pinned version of an in-progress repo resolves to the in-progress node instead of fetching (and recursing into) a divergent — possibly stale-schema — snapshot. `LoadUnified` seeds `loadingRepos[rootIdentity] = merged` (the root's identity comes from its optional `repo:` field, else `git remote origin`), so any transitive import of the root's OWN repo resolves to the local working tree — **the importing project's namespace pins win**. `loadingRepos` entries are pushed before recursing and popped after (stack-scoped — two SIBLING imports of the same repo at different versions still each load); the root seed is never popped. A whole-repo ref with an empty sub-path resolves to that repo's `charly.yml`. Covered by `TestImportNamespace_DivergentVersionMutualCycle` + `TestNsRepoIdentity` in `ns_identity_test.go`.

### Namespace resolver (`charly/namespace.go`)

The resolver implements Go-package-member semantics over `Config.Namespaces`:

- `splitNamespaceRef(ref)` — splits a qualified ref on its FIRST `.` into `(ns, rest)`; a bare ref returns `ok=false`; the remainder may itself be qualified (`a.b.c` → `"a"`, `"b.c"`).
- `resolveBoxRef(ref)` / `resolveLocalRef(ref)` — bare names resolve in the current `Config`; `ns.name` descends into `c.Namespaces[ns]` recursively, returning the entry plus the `Config` (namespace context) it lives in.
- `resolveNamespacedBases(out, …)` — after the local image set resolves, pulls every namespace-qualified `base:` (and qualified `builder:` ref, but only for images that actually have layers to build) into `out`, keyed by fully-qualified name, iterating to a fixpoint (a pulled-in image may reference a deeper namespaced base).
- `pullNamespacedBox(from, ref, keyPrefix, …)` — descends the namespace chain to the leaf, re-keys the entry's own internal base to the fully-qualified ancestor so the build graph references it correctly, and recurses to pull that ancestor.

The inheritance rule lives here: `distro:`/`build:` are VALUES → inherited across a namespace boundary; `builder:` is a map of namespace-relative REFS → NOT inherited (the consumer declares its own). See the file header comment for the rationale (avoid leaking a base-namespace-relative ref into a consumer where that namespace doesn't exist). `leafName(ref)` strips every namespace prefix to the final member name (`arch.arch-builder` → `arch-builder`); paired with `resolveBoxRef`'s returned namespace `Config` it keys the resolved entity in that `Config.Box` map (used by the reachability walk below).

### Remote-layer resolver (`charly/refs.go` + `charly/layers.go`) — per-entity version + reachability-scoped collection

`@github` layer refs resolve in TWO phases: the `:vTAG` git tag is only the FETCH coordinate (which commit to clone); the layer's own `version:` field — read AFTER fetch — is the authoritative identity that drives dedup + warn-and-newest-wins.

- **`CandyRef` (`charly/refs.go`)** — the single representation of a `require:` / `candy:` ref. It stores the ORIGINAL ref string (`Raw`, with any `@repo` prefix and `:version` suffix); `.Bare()` (the map-key form), `.Version()` (the pinned git tag — the FETCH coordinate, NOT the identity), and `.IsRemote()` are DERIVED. A `resolved` slot carries the qualified sibling key set by `qualifyRemoteSiblingDeps` after a remote layer is fetched, so ONE list serves both the graph (keys on `.Bare()`) and the transitive fetch (keys on the immutable `.Raw`). `Candy.Require` / `Candy.IncludedCandy` are `[]CandyRef` — there are no parallel bare/raw arrays.
- **Two-phase per-entity-version resolution** — `CollectRemoteRefsOpts` (`charly/refs.go`) collects EVERY distinct `(repo, git-tag)` a bare ref is referenced at — it does NOT collapse to one winning tag and does NOT warn (the git tag is just where to clone from). The `ScanAllCandyWithConfigOpts` fix-point (`charly/layers.go`) fetches each `(repo, git-tag)` (tracking scanned `(repo,git-tag,ref)` triples), reads each materialization's per-entity `version:`, accumulates candidates per bare ref, then `pickCandyVersion` arbitrates: **same per-entity version across different git tags → NO warning**, the newest git tag wins for freshness (`compareSemver`); **different per-entity versions → warn once** (naming both per-entity versions + sources) and the newest per-entity version wins (`compareCalVer`). Exactly one materialization per bare ref reaches the layer map, so the graph + intermediates are unchanged. A fetched layer with NO `version:` is a HARD ERROR (no fallback — first-party remotes are backfilled by remote-cache auto-migration, `EnsureRepoDownloaded` → `RunProjectMigrations`). `pickCandyVersion` is the SOLE arbiter for direct AND transitive refs, so a transitive dep can never silently pull a different version of an already-resolved layer. **This is why a repo re-tag of an UNCHANGED layer no longer warns** — the old resolver compared the repo git tag, which advances on every push.
- **Reachability-scoped collection (`CollectRemoteRefsOpts.collectBox`)** — collection walks ONLY the enabled root images + the namespaced images reachable via their `base:`/`builder:` edges (`resolveBoxRef` + `leafName`), plus local layers' transitive deps. It does NOT scan every image and `kind:local` template of every imported namespace (that over-collection pulled unrelated layers pinned at a different tag — e.g. a namespace's `charly-cachyos` workstation template's `chrome` — and tripped the version policy). Builder edges ARE followed when an image builds (a namespaced `fedora.fedora-builder` is built as an intermediate and needs its `rpmfusion`/`yay` layers); dropping them under-collects ("unknown layer").
- **One unified populator (`populateCandyFromYAML`, `charly/unified.go`)** — both `scanCandy` (discovered-layer-dir path) and `synthesizeInlineCandy` (charly.yml inline path) call it, so they can't drift. The `Has*` predicates (`HasEnv`/`HasPorts`/`HasVolumes`/…) are derived methods; only the filesystem-probe caches (`HasPixiToml`/`HasSrcDir`/…) stay fields.

`charly box reconcile` (see `/charly-build:reconcile`) is the operator tool that aligns the on-disk git-tag pins so every reference of a repo fetches one commit, clearing any residual per-entity-version warning.

### Capabilities — `BoxMetadata` alias + label completeness check

`Capabilities = BoxMetadata` (type alias in `charly/capabilities.go`). `CapabilityLabelMap` lists every field with its OCI label home; `TestCapabilityLabelCompleteness` fails the build if an `BoxMetadata` field lacks a mapping. This invariant keeps `charly deploy from-box` reliable: every field deploy code might consult is readable from a pushed image's labels alone, independent of `charly.yml`.

### Kubernetes target (fifth DeployTarget)

`K8sDeployTarget` (`charly/k8s_target.go`) sits alongside `OCITarget`, `PodDeployTarget`, `LocalDeployTarget`, `VmDeployTarget`. Unlike local/VM targets, K8s doesn't consume install plans — it emits Kustomize manifests directly from `(Capabilities, K8sGenerateOpts, ClusterProfile)` via `GenerateK8sKustomize` (`charly/k8s_generate.go`). The workload-kind heuristic (`selectWorkloadKind`) translates the generic `kind:` enum to Deployment/StatefulSet/DaemonSet/Job/CronJob/Pod based on intent + storage presence.

### VM target (fourth DeployTarget)

`VmDeployTarget` (`charly/deploy_target_vm.go`) executes InstallPlans inside a running VM over SSH. Same IR as LocalDeployTarget, but bash bodies run via `ssh guest 'sudo bash -s'` through an `SSHExecutor` (`charly/deploy_executor_ssh.go`). Ledger writes land on the **guest** filesystem. The `DeployExecutor` interface (`charly/deploy_executor.go`) decouples "how shell commands run" from the target walking logic — `ShellExecutor` + `SSHExecutor` are the two implementations.

`charly deploy add vm:<name>` dispatches through `deploy_add_cmd.go::dispatchNode` → `ResolveTarget` → `VmUnifiedTarget.Add` / `.Del` (no per-kind dispatch function); `deploy_add_cmd_vm.go` carries the VM-only helpers (`deployNestedPodsInGuest`, `buildVmReverseRunner`, `vmNameFromDeployName`). Full architecture + preflight flow lives in `/charly-internals:vm-deploy-target`.

### YAML surface ↔ Go identifier convention

The codebase keeps wire format (YAML keys) and internal names (Go fields/types) in strict symmetry — plural YAML keys get plural Go identifiers, singular get singular. The singular `builder:` / `distro:` / `init:` top-level keys in `build.yml` and `charly.yml` carry singular Go identifiers: `BuilderMap`, `BoxConfig.Builder`, `BuilderConfig.Builder`, `DistroConfig.Distro`, `InitConfig.Init`. The rule: if you change a YAML tag, also rename the Go identifier. Tests enforce this indirectly — struct literals won't compile if they disagree. Note: the OCI **label key** is grouped under `platform.*` / `builder.*` sub-namespaces — see `LabelPlatformDistro`, `LabelPlatformFormat`, `LabelBuilderUse`, `LabelBuilderProvide` in `labels.go`. Label wire-names are decoupled from YAML/Go identifiers by design.

### Kong `default:"withargs"` for parent+leaf commands

Kong normally treats a struct as either a branch (has child `cmd:""` subcommands) OR a leaf (accepts `arg:""` positionals and has a `Run()` method) — not both. When you want both shapes on the same parent command (e.g., `charly check live <image>` runs tests AND `charly check cdp …` dispatches to a subcommand), tag the default child with `default:"withargs"`. Kong then dispatches to that child when the first token doesn't match a subcommand name, passing positional args/flags through.

Two uses in the codebase:
- `charly/config_image.go:14-21` — `BoxConfigCmd.Setup` is the default; `charly config <image>` routes through `BoxConfigSetupCmd` while `charly config mount|status|…` dispatch explicitly.
- `charly/check_cmd.go:22-31` — `TestCmd.Run` is the default; `charly check live <image>` runs declarative tests while `charly check cdp|wl|dbus|vnc …` dispatch explicitly.

Tradeoff: a subcommand name shadows a positional value with the same text. `charly check cdp` always dispatches to the cdp subcommand — if an image is literally named `cdp`, use the explicit `charly check live cdp` form.

### Mode purity: `LoadConfig` must NOT read `charly.yml`

OCI labels are written exclusively from `charly.yml` at `charly box build` / `charly box generate` time. `charly.yml` is deploy-mode state and must never bleed into the baked image. The key guarantee lives in `charly/config.go:LoadConfig` — it calls `LoadConfigRaw` only, with no `MergeDeployOverlay`.

**The rule**: every build-mode command (anything under `charly box …`) calls `LoadConfig`. If you ever re-introduce `MergeDeployOverlay` inside `LoadConfig`, you will silently contaminate OCI labels with whatever is in the user's local `charly.yml` — exactly the bug that made images bake `ports: ["5900:5900","9250:9222"]` from a stale `charly.yml` entry instead of the `charly.yml`-declared `["5900:5900","9222:9222","9224:9224"]`.

Deploy-mode commands (`charly config`, `charly start`, `charly stop`, `charly update`, `charly deploy add`, `charly deploy del`, `charly shell`, `charly cmd`, `charly service`, `charly vm create`, …) read labels via `ExtractMetadata` and then apply the deploy overlay explicitly via `MergeDeployOntoMetadata(meta, dc, instance)`. This split is load-bearing — never collapse it.

**Host-deploy specifics**: `charly deploy add host` is deploy mode (reads both charly.yml and charly.yml), not build mode — despite looking like "install on host, not into an image". The compiler (`BuildDeployPlan` in `install_build.go`) is pure and shared with build mode, but the invocation path reads charly.yml for `add_candy:` and `install_opts:` like every other deploy-mode command.

### InstallPlan IR — the shared intermediate representation

`charly box build`, pod deploys, and local-target deploys all route through a shared IR. Flow:

```
Layer + ResolvedBox + HostContext
    → BuildDeployPlan (install_build.go) [pure]
    → InstallPlan (install_plan.go)
    → DeployTarget.Emit
       ├── OCITarget (build_target_oci.go)          → Containerfile text
       ├── PodDeployTarget (deploy_target_pod.go)   → overlay + quadlet
       ├── LocalDeployTarget (deploy_target_local.go) → local shell execution
       ├── VmDeployTarget (deploy_target_vm.go)     → SSH-wrapped shell in the guest
       └── K8sDeployTarget (k8s_target.go)          → Kustomize base/overlays tree
```

Full reference lives in **`/charly-internals:install-plan`** — go there before touching any of those files. Supporting Go files (ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref, hostdistro, migrate_services_tool) are covered in **`/charly-internals:local-infra`**.

### VM-path architecture

The VM path spans the following module topology:

| File | Role |
|---|---|
| `charly/vm_spec.go` | `VmSpec` + `VmSource` discriminated union (cloud_image / bootc) + `VmChecksum` + `VmNetwork` + `VmSSH` + `VmKeyInjection` |
| `charly/cloud_init_types.go` | `VmCloudInit` + `VmCloudInitUser/File/Network/Mirrors` + `VmCharlyInstall` (auto/scp/url/skip state machine) |
| `charly/libvirt_yaml.go` | `LibvirtDomain` + 30+ sub-types (features, CPU, clock, memory backing, numatune, cputune, devices, seclabel, launch security, resource, sysinfo) — the opencharly YAML-facing shape of the `libvirt:` stanza |
| `charly/libvirt_yaml_bridge.go` | `RenderDomainXML`/`BuildLibvirtDomainXML` pure functions (build a `libvirtxml.Domain` tree, marshal to XML) + `buildDomainDevices` device emission (passt backend, portForward attribute order, virtio-gpu default, SMBIOS credentials, `XMLPassthrough` merge) |
| `charly/qemu_render.go` | `RenderQemuArgv` for direct-QEMU backend |
| `charly/cloud_init_render.go` + `cloud_init_iso.go` | `RenderCloudInit` + `ResolveKeyInjectionChannels` + `composeUsers` (adopt-merge) + `WriteSeedISO` via xorriso/genisoimage/mkisofs |
| `charly/vm_cloud_image.go` + `http_fetch.go` | `BuildCloudImage` pipeline: fetch URL + sha256 sidecar + resize + seed ISO render |
| `charly/charly_install.go` | `EnsureCharlyInVenue` — the GENERIC "copy charly into a running venue" mechanism (container `podman cp` / VM-SSH `scp` / host `install`, all via `DeployExecutor.PutFile`): returns the `charly` invocation command, copying the host `os.Executable()` to a non-`$PATH` `/tmp/charly-<calver>` on absence/older (idempotent, never shadows a packaged charly). Used by the explicit `charly check dbus notify`/`call` paths + nested from-image delegation, so an image need not bake the `charly` layer. `EnsureCharlyInGuest` is the VM-deploy strategy wrapper (auto/scp/url/skip) layered on top |
| `charly/ovmf_paths.go` | `ResolveOvmfPaths` (per-distro OVMF_CODE/VARS paths) + `EnsurePerVmNvram` + `ResolveOvmfForSpec` (bios-sentinel returning empty strings) |
| `charly/schema/vm.cue` + `cue_kind_vm.go` | `#Vm` — the closed CUE schema validating VmSpec + the `#LibvirtDomain`/`#VmCloudInit` subtrees (the Go VM/libvirt validators were deleted; CUE owns it via the per-kind registry) |
| `charly/deploy_executor*.go` | `DeployExecutor` interface + `ShellExecutor` + `SSHExecutor` with `WaitForSSH` + `WaitForCloudInit` |
| `charly/deploy_target_vm.go` | `VmDeployTarget.Emit` |
| `charly/deploy_add_cmd_vm.go` | VM-only deploy helpers (`deployNestedPodsInGuest`, `buildVmReverseRunner`, `vmNameFromDeployName`); `charly deploy add vm:<name>` itself dispatches through `dispatchNode` → `ResolveTarget` → `VmUnifiedTarget.Add` |
| `charly/vm_create_spec.go` + `vm_build.go` | CLI command wiring for `charly vm build/create` reading `kind: vm` entities |
| `charly/libvirt_helpers.go` + `libvirt_yaml_listen.go` | helpers shared by the libvirt YAML bridge + `qemu_render` argv emitter (`VmRuntimeParams`); structured `<listen>` support for `LibvirtGraphics` |

**`unified.go` VM support**: `"vm"` is in the `entityKind` enum, with a `VmDoc` loader struct, a `mergeVmMap` merger, and `"vm"` in `rootShapeKeys` (so `vm.yml` is recognized as a valid entity-shape include file).

Full subsystem references: `/charly-internals:vm-spec`, `/charly-internals:libvirt-renderer`, `/charly-internals:cloud-init-renderer`, `/charly-internals:vm-deploy-target`, `/charly-internals:ovmf`, `/charly-internals:cutover-policy`.

### Self-exec coordination: host → container AND host → host

The `charly` binary self-execs in two distinct directions.

**Host → container** — the host `charly` delegates to a container-baked `charly` via `exec … charly <subcommand>`. Three sites today:
- `charly/notify.go:20` — best-effort desktop notification via in-container `charly check dbus notify`.
- `charly/dbus.go:195` — strict in-container `charly check dbus notify` with gdbus fallback.
- `charly/dbus.go:229` — generic D-Bus call via in-container `charly check dbus call`.

**Host → host** — the test runner spawns the same host `charly` binary as a subprocess to execute cdp/wl/dbus/vnc declarative verbs:
- `charly/checkrun_charly_verbs.go` — the `runCharlyVerb` dispatcher builds `charly check <verb> <method> <image> [args…]` argv and runs it via `exec.CommandContext`, feeding stdout/stderr through the existing matcher pipeline. `findCharlyBinary()` prefers `os.Executable()` so tests invoke the same build that collected them, falling back to `$PATH`.

**The rule:** whenever you rename a subcommand path crossed by any of these self-exec sites, edit the host-side invocation strings AND plan a coordinated rebuild of every image that bakes the `charly` layer (affected images: grep `charly.yml` for `- charly$`). For host→host sites the rebuild doesn't matter — it's the same binary — but the method-name allowlists in `checkrun_charly_verbs.go` must stay in lockstep with the actual `charly check cdp|wl|dbus|vnc` subcommand tree.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build | `task build:charly` | Compile to `bin/charly` and install as Arch package |
| Install | `task build:install` | Install charly as Arch package (uses pre-built binary) |
| Run tests | `cd charly && go test ./...` | Run all tests |
| Run specific test | `cd charly && go test -run TestName ./...` | Run single test |
| Vet | `cd charly && go vet ./...` | Static analysis |
| Format | `cd charly && gofmt -w .` | Format code |

## Project Directory Structure

```
project/
├── bin/charly                     # Built by `task build:charly` (gitignored)
├── charly/                        # Go module (go 1.25.3, kong CLI, go-containerregistry)
│   └── charly.cue             # The binary's embedded default config (//go:embed,
│                              # embed_defaults.go): distro/builder/init/resource
│                              # build vocabulary + the sidecar: template library.
│                              # Parsed by the SAME unified loader as any project
│                              # charly.yml; a project ships none of it.
├── .build/                    # Generated Containerfiles (gitignored)
├── charly.yml                  # Image definitions
├── Taskfile.yml               # Bootstrap tasks only
├── taskfiles/                 # Build.yml, Setup.yml
├── candy/<name>/             # Layer directories (160 layers)
├── plugins/                   # Git submodule (overthink-plugins, 5 plugins, 244 skills)
└── templates/                 # supervisord.header.conf (referenced by init.supervisord.header_file)
```

Submodule convention: `plugins/` is a submodule rooted at the
`overthink-plugins` repo. Clone with `--recurse-submodules` or run
`git submodule update --init` after a plain clone. See
`/charly-internals:skills` for the skill-authoring and sync conventions.

## Source Code Map

### Core Generation

| File | Purpose |
|------|---------|
| `main.go` | CLI entry point (Kong framework). `CLI` struct carries two global path fields: `Dir` (`-C` / `--dir` / env `CHARLY_PROJECT_DIR`) and `Repo` (`--repo` / env `CHARLY_PROJECT_REPO`). When `Repo` is set, `main()` resolves it via `ResolveProjectRepo` and assigns the cache path back into `Dir`; when `Dir` is non-empty (after that resolution), `main()` calls `os.Chdir(Dir)` **before** `ctx.Run()` — one-line intervention that propagates to every `os.Getwd()` call site throughout build-mode commands without requiring per-command plumbing. `--repo` and `--dir` are mutually exclusive (fast-fail). Covered by `TestCharlyDir_FlagChdir`, `TestCharlyDir_Errors`, `TestCharlyRepo_FlagChdir`, `TestCharlyRepo_DirConflict`, `TestCharlyRepo_DefaultExpansion` in `main_dir_test.go` + `main_repo_test.go`. Load-bearing for `charly mcp serve` inside a container where cwd resolves to `/workspace` (the `charly-mcp` layer default) — either bind-mounted with the project, or empty in which case `bootstrapProject()` auto-falls back to the upstream repo. |
| `main_repo.go` | `--repo` resolver. `DefaultProjectRepo = "github.com/overthinkos/overthink"`. `normalizeRepoSpec(spec)` handles four spec shapes: `"default"` literal, bare `owner/repo` (auto-prefix `github.com/` when first segment has no dot), bare `owner/repo@ref`, host-qualified `host.tld/owner/repo[@ref]`. `ResolveProjectRepo(spec)` reuses `EnsureRepoDownloaded` from `refs.go` so the project-repo cache shares `~/.cache/charly/repos/` (override `CHARLY_REPO_CACHE`) with the existing remote-layer cache. Empty version triggers `GitDefaultBranch` resolution. |
| `config.go` | `charly.yml` parsing, inheritance resolution. `BuildFormats` type. `Distro` field. `ResolvedBox.Tags` (union). `SupportsTag()`, `SupportsBuild()` methods |
| `format_config.go` | `DistroConfig` (with per-distro `Formats`), `BuilderConfig` types. `BuildFile` loader struct matches the three top-level sections of `build.yml` (`distro:`, `builder:`, `init:`). `LoadBuildConfigForBox` resolves a single `format_config: build.yml` ref and splits it into `DistroConfig` / `BuilderConfig` / `InitConfig` views. Per-image config resolution with remote ref support |
| `format_template.go` | Go `text/template` rendering engine. Template helpers: `cacheMounts`, `cacheMountsOwned`, `quote`, `default`, `splitFirst`, `replace`, `join`. `InstallContext`, `BuildStageContext` types |
| `layers.go` | Layer scanning, file detection. `CandyYAML` struct (CUE-decoded via `cue_loader.go`; load-time top-level typo-detection via the `rejectUnknownCandyTopLevelKeys` guard — no custom `UnmarshalYAML`). `Task` struct + `Kind()` method (exactly-one-verb). `derivePackageSectionsFromCalamares` is the SOLE package-surface populator: every `distro:` key (bare / versioned / compound) → a per-distro `tagSections` entry (NOT a shared format section — that collapse caused the non-deterministic deb-repo bug); top-level `package:` → `topPackages` (folded at resolve time); arch `aur:` keeps its `aur` format section. `compileSystemPackageSteps` (`install_build.go`) cascades these — see `/charly-internals:install-plan`. |
| `tasks.go` | **All task emission logic** — per-verb emitters (`emitMkdirBatch`, `emitCopy`, `emitWrite`, `emitLinkBatch`, `emitDownload`, `emitSetcapBatch`, `emitCmd`, `emitBuild`), `emitTasks` orchestrator, `stageInlineContent` (content-addressed), `resolveUserSpec`, `taskSubstPath`, `taskUnresolvedRefs`. Adjacent-coalescing (`taskCoalescesWith`). **Shell-quoting helpers:** `shellSingleQuote(s)` for standard `'...'` escaping (used by LABEL values + `emitDownload` env entries) and `shellAnsiQuote(s)` for bash ANSI-C `$'...'` quoting (used by `emitCmd` so multi-line script bodies survive podman's line-oriented Dockerfile parser). **`emitDownload` env rule:** uses `export VAR=val;` (semicolon-terminated) not `VAR=val cmd`, because bash expands `${VAR}` in URL arguments before the cmd-prefix environment is assembled. ~430 lines, single home for install-task codegen. |
| `generate.go` | Containerfile generation. `writeCandySteps` orchestrates per-layer: packages → `emitTasks` (from `tasks.go`) → builders → USER reset. **Package resolution goes through the SAME `resolveCascadePackages` (`install_build.go`) the deploy compiler uses** — ONE distro-specificity cascade for build AND deploy (folds the top-level `package:` base + unions distro tag sections most-specific-first), then renders the primary format's install template; non-primary build formats (`aur`) emit from their own format section. There is no separate build-path Phase1/Phase2 resolution anymore. Config-driven format install, builder stages, and bootstrap from the build vocabulary (`distro:` + `builder:` sections — the embedded default lives in `charly/charly.cue`). **`writeLabels` is called at the END of the final stage (after the final `USER` directive)** — the volatile `LabelDescription` value would otherwise invalidate every downstream RUN/COPY on a baked-plan edit; with LABELs-at-end, only the LABEL steps themselves re-emit (cache preserves all install work). `writeJSONLabel` routes every JSON label value through `shellSingleQuote` so embedded `'` chars in test commands (`awk '{print $1}'`) don't break podman's `key=value` LABEL parser. |
| `validate.go` | All validation rules. `validateCandyTasks` enforces exactly-one-verb, per-verb required modifiers, `var` key rules, path/mode/caps format, `${VAR}` resolution checks. Format/builder validation against config definitions (not hardcoded maps) |
| `version.go` | CalVer computation |
| `scaffold.go` | `new layer` scaffolding (single-layer dir creation with stub `charly.yml`) |
| `scaffold_project.go` | `new project` scaffolding + `charly.yml` mutation helpers (`ScaffoldProject`, `AddBox`, `AddCandyToBox`, `RemoveCandyFromBox`). All YAML round-trips go through the `yaml.v3` Node API so comments + key order are preserved. Tested in `scaffold_project_test.go`. |
| `scaffold_cmds.go` | All Kong command structs for the MCP-first authoring surface: `NewProjectCmd`, `NewBoxCmd`, `BoxSetCmd`, `BoxAddCandyCmd`, `BoxRmCandyCmd`, `BoxFetchCmd`, `BoxRefreshCmd`, `BoxWriteCmd`, `BoxCatCmd`, `CandyCmd` (with `Set` + four `add-{rpm,deb,pac,aur}` aliases), `CandyAddPkgCmd`. Houses `resolveProjectFile()` — the path-traversal guard for `box write` / `box cat`. Houses `detectPkgSection(os.Args)` — the workaround for Kong not exposing "which alias triggered me" to a shared struct. Houses `appendCandyPackages()` with the scaffold's null-`package:` → sequence upgrade. |
| `yaml_setter.go` | `SetByDotPath(path, dotpath, valueYAML)` — generic comment-preserving YAML setter used by `charly box set` and `charly candy set`. Walks `*yaml.Node` trees; creates intermediate mappings on demand; rejects descent into scalars. Tested in `yaml_setter_test.go` (comment preservation, list values, intermediate-mapping creation, scalar-descent error path). |

### Dependency & Graph

| File | Purpose |
|------|---------|
| `graph.go` | Topological sort (layers + images), `ResolveBoxOrder()` |
| `intermediates.go` | Auto-intermediate image computation (trie analysis). `createIntermediate()` inherits `Distro` and `BuildFormats` **from the parent image first**, falling back to `cfg.Defaults.*` only when the parent is external or empty. Inverting this (defaults winning over the explicit parent) mis-tags every arch-rooted intermediate as `build: [rpm]`, so every layer section keyed on `pac:` emits an empty RUN step (symptom: `arch-ssh-client` ships without `direnv` / `gnupg` / `openssh`). Regression guard: `TestComputeIntermediates_InheritDistroFromParent` uses `defaults.Build=[rpm]` but expects arch-rooted intermediates to come out `[pac]`. |

### Build & Runtime

| File | Purpose |
|------|---------|
| `build.go` | `build` command (sequential image building, retry logic) |
| `merge.go` | `merge` command (post-build layer merging) |
| `shell.go` | `shell` command (execs engine run) |
| `start.go` | `start`/`stop` commands |
| `status.go` | `status` command (structured table/detail view, live tool probing, `--json`) |
| `commands.go` | `enable`/`disable`/`logs`/`update`/`remove` |
| `service.go` | `service` command (init system service management inside containers) |
| `data.go` | Volume data seeding (`provisionData`, `seedKind`, `SeederHelperImage`) for bind-backed + named-volume targets, driven by `charly config --seed`/`--force-seed` |
| `hooks.go` | Lifecycle hooks (`post_enable`, `pre_remove`) collection and execution |
| `remote_image.go` | Remote image ref resolution, pull-or-build |
| `vm.go` | VM lifecycle: create, start, stop, destroy, list, console, ssh |
| `vm_build.go` | VM disk image builds (qcow2, raw via bootc install) |
| `vm_libvirt.go` | Libvirt backend: VM operations via session-level libvirt |
| `vm_qemu.go` | QEMU backend: direct VM operations via qemu-system |
| `smbios_credentials.go` | SSH key injection via SMBIOS/systemd credentials at VM boot |
| `libvirt.go` | Libvirt XML snippet collection and injection |
| `browser_cdp.go` | CDPClient -- lightweight Chrome DevTools Protocol WebSocket client (golang.org/x/net/websocket) |
| `cdp.go` | CDP browser automation commands + `CdpCmd` struct (open, list, close, text, html, url, screenshot, click, type, check, wait). `cdpGetWindowOffset`, `cdpDispatchKeyEvent`, `deepQueryJS` |
| `cdp_spa.go` | SPA-aware remote desktop interaction: `CdpSpaCmd` (click, type, key, key-combo, mouse, status). `spaDetect`, `spaEnsureFocus`, `spaKeyMap`, `spaModifierMap`, coordinate scaling via `--scale` |
| `cdp_test.go` | CDP command + client tests |
| `wl.go` | Wayland desktop commands (screenshot, click, type, key, mouse, status, windows, focus); `SwayCmd` Sway compositor support (`charly check wl sway`) — `swayNode` `Focused`/`FullscreenMode` fields, `searchSwayNode` prefers focused/fullscreen nodes, XWayland class matching via `swayWindowProperties`. `--from-x11` flag + `FindX11WindowGeometry()` for XWayland coordinate translation |
| `vnc.go` | VNC desktop commands (screenshot, click, type, key, mouse, status, passwd, rfb). `--from-x11` flag for XWayland coordinate translation |

### Infrastructure

| File | Purpose |
|------|---------|
| `engine.go` | Docker/Podman abstraction, `ResolveBoxEngineForDeploy()` |
| `registry.go` | Remote image inspection (go-containerregistry) |
| `transfer.go` | Cross-engine image transfer |
| `runtime_config.go` | `~/.config/charly/config.yml`, `secret_backend` key, credential maps |
| `network.go` | Shared "charly" container network management |
| `machine.go` | Podman machine management (rootful VM builds) |

### Configuration

**Key types — user_policy + exclude_distros architecture:**

| Type / Field | File | Purpose |
|---|---|---|
| `DistroDef.BaseUser *BaseUserDef` | `format_config.go` | Pointer to a declared pre-existing uid-1000 account in the upstream base image. Nil when not declared (fedora/arch/debian); set for ubuntu (`{ubuntu, 1000, 1000, /home/ubuntu}`). Inherited via `resolveInherits` so a child distro with no `base_user:` inherits the parent's |
| `BaseUserDef` | `format_config.go` | Four required fields: `Name`, `UID`, `GID`, `Home`. Parsed from `build.yml distro.<name>.base_user:` |
| `BoxConfig.UserPolicy string` | `config.go:130` | YAML field `user_policy`. Values: `auto` (default) / `adopt` / `create`. Drives the reconciliation switch in `ResolveBox` |
| `ResolvedBox.UserAdopted bool` | `config.go:194` | True when the policy reconciliation adopted a distro's `BaseUser` (User/UID/GID/Home overwritten). Consumed by `writeBootstrap` in `generate.go` to skip the useradd step |
| `Op.ExcludeDistros []string` | `checkspec.go` | Per-test filter — test runner in `checkrun.go:runOne` skips the check when any of the image's distro tags intersects with this list. Reason reported as `excluded on distro "<tag>"` |
| `TagPkgConfig.Raw map[string]any` | `layers.go` | Captures the full YAML map for a tag section (e.g. `debian:13:`), not just `package:`. Enables `repos:`, `keys:`, `options:` inside tag sections. Read by the generator's install-template emission path |

**Policy reconciliation flow** (`charly/config.go:ResolveBox`, after distroDef loaded):

```go
policy := img.UserPolicy
if policy == "" { policy = c.Defaults.UserPolicy }
if policy == "" { policy = "auto" }
baseUser := (*BaseUserDef)(nil)
if resolved.DistroDef != nil { baseUser = resolved.DistroDef.BaseUser }
userExplicitlySet := img.User != "" || c.Defaults.User != ""

switch policy {
case "adopt":
    if baseUser == nil { return nil, fmt.Errorf(...) }
    // overwrite User/UID/GID/Home
    resolved.UserAdopted = true
case "auto":
    if baseUser != nil && !userExplicitlySet {
        // overwrite User/UID/GID/Home
        resolved.UserAdopted = true
    }
case "create":
    // no-op
}
```

See `/charly-image:image` "user_policy" for the user-facing decision matrix, `/charly-build:build` "base_user:" for the declarative side, and `/charly-build:generate` "writeBootstrap" for the consumer side.

### Existing configuration files

| File | Purpose |
|------|---------|
| `env.go` | ENV merging, path expansion |
| `envfile.go` | `.env` file parsing (`ParseEnvFile`, `ParseEnvBytes`), runtime env var resolution/merging |
| `security.go` | Container security config collection, CLI args generation. Merges `Mounts` from layer security configs |
| `labels.go` | OCI label constants. `LabelDescription` (`ai.opencharly.description`) carries the `LabelDescriptionSet` — each `LabeledDescription` (a `Description` string) plus its `Plan []Step` list; `BoxMetadata`'s `*LabelDescriptionSet` field is populated by `ExtractMetadata` when present |
| `egress.go` | **Egress validation** — `ValidateEgress(kind,label,bytes)` gates the config files charly WRITES (cloud-init, k8s, units, ssh_config, …) against a CUE schema before the bytes hit disk. VENDORED schemas (`schema/vendor/*.cue`, package+import) compile as their own `cue.Value` via `registerVendoredEgressKind` (they can't join the package-less concatenated `sharedCueSchema`); charly's own package-less egress schemas resolve via `cueKindDef`. See `/charly-internals:egress`. |
| `volumes.go` | Named volume collection/mounting |
| `alias.go` | Command aliases (wrapper scripts) |
| `deploy.go` | Per-deployment config overlay, `DeployVolumeConfig`, `ResolveVolumeBacking()`, `saveDeployState()`, `cleanDeployEntry()` (instance-aware provides cleanup) |
| `provides.go` | Env/MCP provides injection, `removeBySource()`, `removeByExactSource()` (instance-specific cleanup), `podAwareMCPProvides()` |
| `enc.go` | Encrypted volumes (gocryptfs via `systemd-run --scope --user --unit=charly-enc-<image>-<volume>`), `ResolvedBindMount`. `-allow_other` required for rootless podman keep-id. `encUnmount()` stops scope units after fusermount. Stale scope retry on mount failure |
| `devices.go` | Host device auto-detection. `DetectedDevices` struct with `RenderNode` field (first `/dev/dri/renderD*`). `appendAutoDetectedEnv()` centralizes injection of `HSA_OVERRIDE_GFX_VERSION`, `DRINODE`, `DRI_NODE` — called at 10 sites across config_image.go, start.go, shell.go. Uses `appendEnvUnique` so user `-e` flags always override |
| `tunnel.go` | Tunnel providers (Tailscale, Cloudflare), backend scheme helpers (`schemeTarget`, `tailscaleFlag`, `isTCPFamily`), start/stop for each provider |
| `quadlet.go` | Quadlet .container file generation, `Secret=` directives |
| `credential_store.go` | `CredentialStore` interface, `ResolveCredential()`, `DefaultCredentialStore()`, `ConfigMigrateSecretsCmd` |
| `credential_keyring.go` | System keyring backend (`go-keyring`: GNOME Keyring, KDE Wallet, KeePassXC via FdoSecrets / Secret Service) |
| `credential_config.go` | Config file credential backend (plaintext fallback for headless) |
| `migrate_secrets_kdbx.go` | `charly migrate` — strips residual `secret_backend: kdbx` + `secrets_kdbx_*` keys from `~/.config/charly/config.yml` |
| `secrets.go` | Container secret collection from labels, Podman secret provisioning, `SecretArgs()` |
| `secrets_cmd.go` | `charly secrets` CLI commands (list, get, set, delete, import, export — all retargeted to `DefaultCredentialStore()`) + the `gpg` subgroup |
| `secrets_gpg.go` | `charly secrets gpg` commands (show, env, edit, encrypt, decrypt, set, unset, add-recipient, recipients) |

### Remote Layer Refs

| File | Purpose |
|------|---------|
| `refs.go` | Remote ref types, parsing, cache management. `CHARLY_REPO_OVERRIDE` (`RepoOverrideEnv`) Go-`replace`-style local-tree override (`repoOverrideDir`). `selfSuperprojectOverridePair(dir)` derives the bed project's OWN superproject override (`git rev-parse --show-superproject-working-tree` → `rootRepoIdentity`); `mergeRepoOverrides` appends it after operator entries (operator wins). `runCheckBed` auto-applies it so a `box/<distro>` bed tests LOCAL parent-repo candies, never the pinned remote — the candy-ref analogue of auto `--dev-local-pkg`. Tests: `repo_override_test.go`. |
| `refs_git.go` | Git operations: clone, resolve ref, tag resolution |

### Declarative Testing

Implements the `charly check live` / `charly check box` commands and the
`ai.opencharly.description` OCI label. User-facing authoring, verb catalog,
runtime variables, and charly.yml overlay rules live in `/charly-check:check` — this
section is the Go-implementation map.

| File | Purpose |
|------|---------|
| `checkspec.go` | `Op` struct (the unified verb vocabulary — the former Task + Check merged into one; `Kind()` enforces exactly-one verb). Built-in verbs (file/port/command/http/package/service/process/dns/user/group/interface/kernel-param/mount/addr/matching) plus 9 live-container verbs (`cdp`/`wl`/`dbus`/`vnc`/`mcp`/`record`/`spice`/`libvirt`/`k8s`) dispatched via `checkrun_charly_verbs.go`. **`Status` on the http verb is a plain `int`** — not a MatcherList. One expected code per test; no `[200, 302]` list shorthand. `Matcher` + `MatcherList` with custom YAML **and** JSON unmarshalers for scalar/list/map shorthand — symmetry between charly.yml authoring and hand-crafted OCI labels. The plan step types travel in `LabelDescriptionSet` (the `ai.opencharly.description` label carries a `Plan []Step` field per `LabeledDescription`). Extended `${NAME[:arg]}` matcher (in `ExpandTestVars`) — backward-compatible widening of `taskVarRefPattern` in `tasks.go`. **No bash-style defaults**: `${VAR:-fallback}` is unsupported; only `${IDENT}`. `ExpandTestVars`, `TestVarRefs`, `IsRuntimeOnlyVar`, `Op.ExpandVars`. |
| `checkvars.go` | `ResolveCheckVarsBuild` / `ResolveCheckVarsRuntime`. `InspectContainer` is a swappable package-level `var` (test-friendly pattern matching `InspectLabels` in `labels.go`). Maps `podman inspect` output into `HOST_PORT:<N>`, `VOLUME_PATH:<name>`, `VOLUME_CONTAINER_PATH:<name>`, `CONTAINER_IP`, `CONTAINER_NAME`, `ENV_<NAME>`. |
| `checkrun.go` | `Runner`, `Executor` interface, `ContainerExecutor` (via `podman exec`), `ImageExecutor` (via `podman run --rm`). `CheckStatus`/`CheckResult` types (named to avoid collision with doctor.go). Per-verb dispatch for `file`/`port`/`command`/`http`. Matcher evaluation: `matchOne` + `matchNumeric` (`lt`/`le`/`gt`/`ge`). `validMatcherOps` allowlist kept in lockstep with the runner switch by `TestMatcher_AllowlistRunnerSync`. Output formatters: text, JSON, TAP. |
| `checkrun_verbs.go` | Dispatch for the remaining verbs: `package` (rpm/dpkg/pacman), `service` (supervisorctl + systemctl), `process` (pgrep), `dns` (host-side `net.LookupIP` or in-container `getent`), `user`/`group` (getent passwd/group), `interface` (`ip -o addr show` + MTU), `kernel-param` (`sysctl -n`), `mount` (`findmnt`), `addr` (host-side `net.DialTimeout` or in-container `nc -z`), `matching` (pure in-process value matching). **`resolvePackageName(c, distros)`** implements the distro-aware package-map: when `Check.PackageMap` is non-empty, the first entry in `Runner.Distros` that matches a key wins; otherwise `Check.Package` is used as-is. Covered by `TestResolvePackageName` (6 sub-cases including empty-map fallback, first-matching-tag-wins priority, and empty-string-map-value fall-through). `Runner.Distros` is populated from `meta.Distro` at both entry points in `check_cmd.go`. |
| `check_peer.go` | **Cross-deployment probing** — a DRIVER deployment probing a SEPARATE SUBJECT. `liveTargetResolver` is the `on:` TargetResolver for `charly check live` + beds (resolves a driver via `resolveCheckVenue` + `ResolveCheckVarsRuntime`); wired into `CheckLiveCmd.Run` (pod path) AND `runVm` (VM path). `${PEER_HOST:name}` / `${PEER_ENDPOINT:name:port}` address vars (`applyPeerVars` → `collectPeerRefs` → `resolvePeerVars`) are pre-resolved into `Runner.PeerVars`, overlaid by `effectiveEnv` onto whatever resolver is active (primary / `on:`-swapped / harness — one injection point). `PEER_HOST` = the subject's `charly-<name>` container DNS (via `resolveContainer`, which also verifies running); `PEER_ENDPOINT` = host-reachable `resolveCheckEndpoint`. Registered runtime-only in `checkspec.go` `runtimeOnlyVarPrefixes`. |
| `deploy_peers.go` | **`peer:` sibling lifecycle** (shared by check + deploy — R3). `foldPeers` registers each `DeploymentNode.Peer` entry as a top-level addressable Deploy entry (`PeerOf` set, disposability inherited); `validatePeers` enforces dot-free + valid-target peer keys. `bringUpPeers` / `tearDownPeers` shell out (via the package-var `runCharlySubcommand`) to `charly config`+`charly start` (pod peers) / `charly deploy add`+`del` (other), invoked by `DeployAddCmd`/`DeployDelCmd` (operator) AND the bed runner (`check_bed_run.go`, pod + VM paths). Peers are excluded from `bedCheckLiveRefs` (instruments, never check-live'd). |
| `checkrun_charly_verbs.go` | Dispatch for the nine live-container verbs (`cdp`/`wl`/`dbus`/`vnc`/`mcp`/`record`/`spice`/`libvirt`/`k8s`). Hand-enumerated method allowlists (`cdpMethods`/`wlMethods`/`dbusMethods`/`vncMethods`/`mcpMethods` etc.) map each method name to its `charly check <verb> <method>` subcommand path, required modifier fields, and positional-arg builder. The `runCharlyVerb` dispatcher handles skip (RunModeBox → skip with message; empty `r.Box` → skip), required-modifier enforcement (via `checkRequiredFields` + `isZeroField`), subprocess exec via `findCharlyBinary()`, matcher pipeline through `matchAll`, and post-run `artifact_min_bytes` size assertions for screenshot methods. |
| `mcp.go` | `charly check mcp …` Kong subcommand tree. `McpCmd` parent + 7 leaves (`ping`, `servers`, `list-tools`, `list-resources`, `list-prompts`, `call`, `read`). Each leaf resolves the image's `mcp_provide` metadata via `ExtractMetadata`, opens a `*mcp.ClientSession` through the swappable `mcpOpenSession` helper, and exercises one SDK operation. Output is tab-separated plaintext by default; `--json` emits the SDK's native result struct. Uses `github.com/modelcontextprotocol/go-sdk/mcp` v1.5.0. |
| `mcp_client.go` | SDK wrapper layer: `resolveMCPEntry` / `pickMCPEntry` (disambiguator), `resolveContainerNameTemplate` ({{.ContainerName}} substitution), `rewriteMCPURLForHost` (container-network hostname → `127.0.0.1:<published-host-port>` via the same `NetworkSettings.Ports` data that `mergeRuntimeVars` in `checkvars.go:200-213` reads for `HOST_PORT:N`), `buildMCPTransport` (http → StreamableClientTransport, sse → SSEClientTransport), plus formatters (`formatTool` / `formatResource` / `formatPrompt` / `extractToolText`). The rewriter is the load-bearing piece that keeps MCP URLs reachable from the host without authors having to carry separate "internal" vs "external" URL declarations. |
| `mcp_server.go` | **`charly mcp serve`** — turns the entire charly CLI into an MCP server. `McpCmdGroup` + `McpServeCmd` (flags: `--listen`, `--path`, `--stdio`, `--read-only`, `--no-default-repo`). `buildMcpServer(readOnly)` constructs a fresh `kong.New(&modelCLI)`, walks `k.Model.Leaves(true)` to enumerate every leaf command, and calls `server.AddTool(kongLeafToTool(...), makeToolHandler(...))`. Result: **190 tools** auto-generated from Kong struct tags with zero hand-written schema — `--long` flags become properties, positionals become required properties, `enum:"..."` surfaces as JSON-schema `enum`, `default:"..."` as `default`, and every schema has `additionalProperties: false` (LLM-honest: unknown keys are rejected by the SDK's input validation before the handler runs). The `mcpDestructivePaths` map flags **63 entries** with `DestructiveHint: true` (including the MCP-first authoring verbs that mutate `charly.yml` / write files); `--read-only` filters these out at registration time (not runtime gating). `bootstrapProject()` runs before the server starts: chains through env vars (`CHARLY_PROJECT_DIR`, `CHARLY_PROJECT_REPO`) → local `charly.yml` → auto-fallback to `overthinkos/overthink`; `--no-default-repo` opts out of the fallback. The env-var check is the only way to detect parent-flag state from a subcommand receiver (Kong does not expose `CLI` upward). Tool invocation goes through `captureAndRun(argv)` which redirects **os.Stdout / os.Stderr** (package-level vars, not fd-level via `syscall.Dup2` — see the commentary block for why: the SDK's stdio transport captures `os.Stdout` by pointer at Connect time, and dup2 would silently route JSON-RPC responses into the tool-output buffer). Transport: `mcp.NewStreamableHTTPHandler` for HTTP mode, `&mcp.StdioTransport{}` for stdio. `runMu` serialises calls because os-level stream redirects are global. Test coverage: `mcp_server_test.go` (schema presence, destructive-hint annotation, `--read-only` filter — currently 127 read-only + 63 destructive = 190 — positional/flag schema shape, enum/default surfacing, `additionalProperties: false`, `TestMcpServer_VersionRoundTrip` round-trip) + `mcp_serve_default_repo_test.go` (auto-fallback behaviour, hermetic via `CHARLY_REPO_CACHE` pre-seeding). |
| `main_dir_test.go` | Integration tests for the `-C` / `--dir` / `CHARLY_PROJECT_DIR` global: spawns a freshly-compiled `charly` binary from `/tmp` with a scratch project, verifies all three flag forms make `charly box list boxes` resolve the scratch `charly.yml`. Error cases: missing dir, file-not-dir. |
| `local_image.go` | `resolveLocalImageRef(engine, input)` — test-mode-only image resolution that never reads `charly.yml`. Full refs pass through with a `LocalImageExists` check; short names match against `ListLocalImages()` output using label-preferred matching (`ai.opencharly.box=<name>`) with a repo-name trailing-component fallback. Returns `ErrImageNotLocal` on no-match so `FormatCLIError` renders the "charly box pull / charly box build" recommendation. Used by `CheckBoxCmd.Run()` to keep `charly check box` purely OCI-labels-driven. |
| `description_collect.go` | `CollectDescriptions(cfg, layers, imageName) *LabelDescriptionSet` walks the base-image chain — mirror of `CollectHooks` in `hooks.go:18-68` — with a visited-image guard so pathological cycles reported by `validateBoxDAG` can't hang the collector. Bucketizes plan steps into `candy`/`box`/`deploy` by source + context, stamps `Origin` for reporting. `MergeDeployDescriptions(baked, local)` implements id-based replace, append, and `{id: X, skip: true}` disable semantics. |
| `check_cmd.go` | `CheckCmd` — the top-level `charly check` command tree with three primary verbs (`Box CheckBoxCmd`, `Live CheckLiveCmd`, `Run CheckRunCmd`) plus 9 live-container probe sub-Cmds (cdp/wl/dbus/vnc/mcp/record/spice/libvirt/k8s) plus the check-run management subcommands (list-agent/list/sync-credential/report/scope/last-tag/note/run-local/self-evaluate). `charly check live` flow: `resolveContainer` → `containerImageRef` → `ExtractMetadata` → load `DeploymentNode.Plan` overlay → `MergeDeployDescriptions` → `ResolveCheckVarsRuntime` → populate `Runner.Box`/`Instance` (so `cdp`/`wl`/`dbus`/`vnc` verbs can build CLI invocations) → `Runner.Run` → format results. `charly check box` flow: `resolveLocalImageRef` (never reads charly.yml — see `local_image.go`) → `ExtractMetadata` → `ResolveCheckVarsBuild` → disposable `ImageExecutor` → run ONLY the build-context `check:` steps (deploy/runtime steps skipped — for full-stack live check use `charly check live <name>`). |
| `check_runner_cmd.go` | The check-run management Cmds: `CheckRunCmd`, `CheckRunLocalCmd`, `CheckListAgentCmd`, `CheckListRunsCmd`, `CheckSyncCredCmd`, `CheckScopeCmd`, `CheckLastTagCmd`, `CheckSelfCheckCmd`, `CheckReportCmd`, `CheckNoteCmd`. The orchestrator's preflight (`runWithPhaseResync`) restarts the disposable harness sandbox (the `iterate.sandbox:` target), syncs credentials, and dispatches `charly check run-local` inside it via `podman exec`. |
| `check_loop.go` | The AI iteration loop core: per-iter dispatch, scoring, plateau bookkeeping, watchdog integration, NOTES.md memory, `commitIterationBestEffort` (where the orphan-bash defense kills issue-52328 deadlock leftovers between iterations), result-file emission. |
| `check_runner_live.go` | `RunCheckLive` — the live step-by-step probe driver invoked at iter end by the harness scorer. Buckets `check:` steps by `pod:`, resolves chains via `ResolveDeployChain` for dotted paths, dispatches to the right `DeployExecutor`. Same code path used by `charly check self-evaluate` (the AI-side mid-iter sanity check). |
| `check_watchdog.go` | `ProgressWatchdog` — per-iteration scoring-progress monitor. Every `progress_check_interval` (default 5m), runs `RunCheckLive` against in-scope `check:` steps, records a `WatchdogSample`, emits a `harness: progress [phase X/N iter Y] elapsed Nm — current score A/B` stderr line. Cancels the AI runner's context if `progress_no_improvement_timeout` (default 30m) of zero score delta passes. |
| `validate_check.go` | `validateOps(cfg, layers, errs)` hooked into `Validate` in `validate.go` (per-op checks via `validateCheck(c *Op, …)`). Enforces: exactly-one-verb per Op, attribute types, port range (1-65535), `time.Duration` parse on `timeout`, `context` ∈ {build,deploy,runtime}, build-context steps can't reference runtime-only variables (via `IsRuntimeOnlyVar`), `id:` uniqueness per section (including cross-layer collisions via `validateCollectedIDUniqueness` → `CollectDescriptions`), matcher-op allowlist (kept in lockstep with `matchOne`), per-verb method-allowlist and required-modifier checks for `cdp`/`wl`/`dbus`/`vnc`/`mcp` (via `validateCharlyVerb` — deploy-context-only enforcement, method validation against `cdpMethods`/`wlMethods`/`dbusMethods`/`vncMethods`/`mcpMethods` maps in `checkrun_charly_verbs.go`). |

**Related skill**: `/charly-check:check` is the authoring-facing reference.

## Go Module Info

- Go version: 1.25.3
- Key dependencies: `kong` (CLI), `go-containerregistry` (OCI), `go-keyring` (Secret Service API)
- Module path: `charly/go.mod`

## Common Workflows

### Add a New CLI Command

1. Define command struct in appropriate file (or new file)
2. Add to CLI struct in `main.go`
3. Implement `Run()` method
4. Add tests in `*_test.go`
5. Build and test: `cd charly && go test ./... && go build -o ../bin/charly .`

### Add a New Validation Rule

Add to `charly/validate.go`. All validation rules are centralized there.

### Updating Go code when an ingress CUE schema changes

The ingress CUE schemas (`charly/schema/*.cue`) and the Go structs they
validate-then-decode-into are TWO hand-maintained sources that MUST stay in
lockstep — there is no codegen between them (see point 5). When you add or
change a `schema/*.cue` definition, update the Go side in the SAME change:

1. **Match the Go struct's yaml tag to the schema field key.** The schema key is
   the WIRE key — e.g. CUE `#InstallOpts.with_service?` pairs with the Go field
   `WithServices` carrying the struct tag `yaml:"with_service"`. The wire key is
   SINGULAR even though the Go field and the `--with-services` CLI flag are
   plural. A mismatch is a SILENT bug, not a compile error:
   - a schema field with NO matching Go yaml tag → CUE accepts the key but the
     loader's yaml.v3 decode silently DROPS the value (`keylint.go` surfaces it
     only as a non-fatal `key ignored` warning);
   - a Go yaml-tag field with NO matching schema field → CUE hard-rejects an
     otherwise-valid config with `field not allowed` (a false positive).
2. **Keep the definition CLOSED.** A `#Def: {…}` is closed by DEFAULT (rejects
   unknown keys — that is exactly what catches a misspelled field). For two
   mutually-exclusive fields use a disjunction applied with `&`
   (`#Box: {…} & ({from?: _|_} | {base?: _|_})`), which keeps the struct CLOSED;
   NEVER an embedded `matchN`, which silently disables closedness (the comments
   in `box.cue` / `candy.cue` / `vm.cue` document this exact choice).
3. **New kind:** add `schema/<kind>.cue` (the `#<Kind>` def, reusing the shared
   defs in `_common.cue`) + a one-line `cue_kind_<kind>.go` registration (mirror
   an existing one; call `registerCueKind`), and add the kind to the corpus test
   (`cue_kinds_corpus_test.go`).
4. **Run the guards before committing:** `cd charly && go test ./...` (the
   corpus + closedness tests and `TestEmbeddedDefaults_SchemaConformance` fail on
   a schema↔struct divergence) and `charly box validate` across the project +
   every `box/<distro>` submodule (the embedded `charly.cue` vocabulary is
   covered by `validateVocabularyCollections`). Add or extend a test for every
   new field set — the test is the regression guard that keeps the two sources
   from drifting.
5. **Do NOT generate the Go structs with `cue exp gengotypes`.** It generalizes
   every disjunction to `any` (so the closed `& ({a}|{b})` patterns and any
   `string | int` field lose their Go-side type precision) and remaps names
   (`#foo`→`Foo`), so it cannot reproduce the hand-written loader structs. The
   manual lockstep + the `go test` guards above are the single-source-of-truth
   enforcement instead.
6. **`keylint` self-decoding types:** a type whose authored keys are NOT literal
   struct fields (the operator-map matchers, the dynamic per-shell sub-blocks) is
   false-flagged by yaml.v3 `KnownFields(true)`; add it to
   `keylintSelfDecodingTypes` in `keylint.go`. CUE's closed-schema check is the
   real unknown-key gate for these (see the `keylint.go` header). The ingress
   recipe is owned by `/charly-build:validate`; the egress analog is the "Adding
   a new egress schema" recipe in `/charly-internals:egress`.

### Debug a Build Issue

```bash
# Generate Containerfiles without building
bin/charly box generate

# Inspect generated output
cat .build/<image>/Containerfile

# Validate configuration
bin/charly box validate

# Inspect resolved image config
bin/charly box inspect <image>
```

### Intermediate image cache invalidation

`charly box build` auto-generates intermediate images (e.g., `ghcr.io/overthinkos/charly-fedora-2-dbus-nodejs`) that bundle the `charly` layer plus common layers for cache reuse across many downstream images. These intermediates are aggressively podman-cached. Updating `candy/charly/bin/charly` does invalidate the COPY step inside the intermediate, but if the intermediate tag already exists locally, `charly box build` may reuse it without re-running the build chain. To force a fresh binary propagation after a manual `bin/charly` update:

```bash
charly clean --invalidate 'charly-fedora-2*'
charly box build <image>
```

This also interacts with the dual-path gotcha documented in `/charly-tools:charly`: `bin/charly` (repo-root, used by host-side invocations) and `candy/charly/bin/charly` (what the `charly` candy actually copies into images) must stay in sync. The canonical `task build:charly` path does both; a manual `go build -o bin/charly ./charly` needs an explicit `cp bin/charly candy/charly/bin/charly` follow-up.

## Implementation insights

These are hard-won lessons that shape the Go-side architecture. They're not obvious from reading the source cold; skim this list before making structural changes.

### Kong flag-namespace collision

Top-level flags and subcommand flags share one global namespace. Declaring `Repo` on both `CLI` (`charly/main.go`) and `McpServeCmd` (`charly/mcp_server.go`) panics with `duplicate flag --repo` at Kong parse time. Resolution: drop the subcommand flag entirely; users write `charly --repo … mcp serve`. Only keep subcommand flags when they have no top-level twin (e.g. `--no-default-repo` on `McpServeCmd` has no collision and stays).

### Env-var proxy for parent-flag detection

From inside a `McpServeCmd.Run()` receiver, you can't reach the parent `CLI` struct. To detect "did the user set the top-level `--dir` / `--repo`" at command-run time, read `os.Getenv("CHARLY_PROJECT_DIR")` / `os.Getenv("CHARLY_PROJECT_REPO")`. Kong populates env vars from flags before `main()` runs, so the env is a reliable proxy whether the user passed the flag or the env var. Implementation: `McpServeCmd.bootstrapProject()` at `charly/mcp_server.go`.

### `detectPkgSection(os.Args)` — sharing one struct across four Kong aliases

Kong dispatches four distinct subcommand aliases (`add-rpm`, `add-deb`, `add-pac`, `add-aur`) to the same `CandyAddPkgCmd` struct. Kong does **not** expose "which alias triggered me" to the struct's `Run()` method. Workaround: derive the section name from `os.Args` (`charly/scaffold_cmds.go`'s `detectPkgSection`). Ugly but contained — the alternative is four almost-identical structs + four dispatchers.

### `yaml.v3` Node API is the single reason edits preserve comments

Unmarshal-to-value + re-marshal scrambles comments, key order, and node styles. Every `charly.yml` editor in the authoring surface (`SetByDotPath` in `charly/yaml_setter.go`; `AddBox` / `AddCandyToBox` / `RemoveCandyFromBox` in `charly/scaffold_project.go`; `appendCandyPackages` in `charly/scaffold_cmds.go`) navigates `*yaml.Node` trees directly and only serializes with `yaml.Marshal(root)` at the very end. Tests (`charly/yaml_setter_test.go`, `charly/scaffold_project_test.go`) explicitly verify that leading file comments, sibling keys, and per-key inline comments all survive round trips.

### Scalar-to-sequence upgrade (scaffold `package:` null)

The layer scaffold writes `rpm:\n  packages:\n  # Add RPM packages here\n` — the value of `package:` parses as scalar-null, not a sequence. Naively calling `candiesNode.Content = append(...)` silently no-ops. `appendCandyPackages` (`charly/scaffold_cmds.go`) checks `pkgsNode.Kind != yaml.SequenceNode` and upgrades in place (`Kind = yaml.SequenceNode; Tag = "!!seq"; Value = ""; Content = nil`). This preserves the key+comment association on serialization. Any other "upgrade a null scalar to a collection" path needs the same pattern.

### Path-traversal guard on the `box write` / `box cat` escape hatch

`resolveProjectFile(projectDir, relPath)` in `charly/scaffold_cmds.go` is the single safety boundary for agent-driven file writes. It rejects absolute paths, calls `filepath.Clean`, then uses `filepath.Rel` + a prefix check to confirm the result stays inside the project root. Any future "free-form file read/write" verb must go through the same helper.

### Project-dir resolver is a two-step resolver, not one

`charly/main.go` resolves the project dir in two steps: `--repo` resolves to a cache path first (`charly/main_repo.go` calls `ResolveProjectRepo` → `EnsureRepoDownloaded`), then falls through into the `os.Chdir(cli.Dir)` block. The two paths are mutually exclusive (fast-fail if both are set). Downstream code just reads `os.Getwd()` — no per-command plumbing. Tested in `charly/main_repo_test.go` (hermetic via `CHARLY_REPO_CACHE` pre-seeding) + `charly/mcp_serve_default_repo_test.go`.

## R9 — deployed binary matches source; runtime deps live in the PKGBUILD

CLAUDE.md R9, operationalized for the `charly` toolchain:

- **Syncing source does not rebuild the binary.** Syncthing / git / rsync move
  *source* between hosts. After pushing code, rebuild on the target
  (`task build:charly`) and verify `charly version` matches what you built — if the
  version is old, the fix under test isn't really under test. The freshness
  guard (above) catches a stale `/usr/bin/charly` against newer `charly/*.go`, but the
  version check is still the explicit proof.
- **Every runtime OS dependency goes into `pkg/arch/PKGBUILD` `depends=`** —
  the single source of truth (`nc`, `socat`, `xorriso`, `qemu-guest-agent`, …);
  the `pkg/fedora` / `pkg/debian` packaging mirrors it. A manual install on one
  host is a bug report disguised as a fix — it won't survive a fresh install on
  a synced host.

The verification side (checking the deployed binary + deps on a live target)
is `/charly-check:check` Standards 7–8; the dual-path `bin/charly` ↔
`candy/charly/bin/charly` gotcha is above and in `/charly-tools:charly`.

## Style Guide

- All logic belongs in Go. Taskfiles are only for bootstrap (building charly).
- Taskfiles for bootstrap only, Go for all other logic.
- Test files alongside source files (`foo.go` -> `foo_test.go`).

## Cross-References

- `/charly-internals:generate-source` — Understanding generated Containerfiles + deep dive on the task emission pipeline (`charly/tasks.go`).
- `/charly-image:layer` — **Canonical author-facing reference** for the task verb catalog that `charly/tasks.go` implements.
- `/charly-build:validate` — Validation rules and error handling (`validateCandyTasks` in `charly/validate.go`).
- `/charly-build:build` — Using the built CLI.
- `/charly-check:check` — Author-facing reference for the declarative-testing feature that `checkspec.go` / `checkvars.go` / `checkrun.go` / `checkrun_verbs.go` / `checkrun_charly_verbs.go` / `description_collect.go` / `check_cmd.go` / `local_image.go` / `validate_check.go` / `mcp.go` / `mcp_client.go` implement.
- `/charly-build:charly-mcp-cmd` — Author-facing reference for both (a) the `charly check mcp` client verb (method catalog, URL-rewrite behavior, port-publishing gotcha, transport dispatch — pair with the file table's `mcp.go` + `mcp_client.go` rows above) and (b) the `charly mcp serve` server (190 tools auto-generated from Kong reflection including the MCP-first authoring surface, destructive-hint + `--read-only` filter, Streamable-HTTP + stdio transports, auto-fallback to `overthinkos/overthink` — pair with `mcp_server.go` + `main_repo.go` + `scaffold_cmds.go` + `scaffold_project.go` + `yaml_setter.go` above).
- `/charly-coder:charly-mcp` — The candy that deploys `charly mcp serve` inside a container: bind-mount volume NAME `project` at the container PATH `/workspace`, `CHARLY_PROJECT_DIR=/workspace` so build-mode MCP tools (`box.list.boxes`, `box.inspect`, etc.) reach `charly.yml` from outside the project checkout — or auto-fall back to `overthinkos/overthink` when `/workspace` is empty (the fallback fires on absence of charly.yml, not absence of CHARLY_PROJECT_DIR).
- `/charly-check:cdp`, `/charly-check:wl`, `/charly-check:dbus`, `/charly-check:vnc` — the four sibling live-container verbs.
- Source: `charly/` directory (~79 source + ~55 test .go files).

## When to Use This Skill

**MUST be invoked** before reading or modifying Go source files. Invoke this skill BEFORE launching Explore agents on charly/ code.

## Live-deploy verification is mandatory (see `/charly-check:check` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/charly-internals:disposable`). Use `charly update <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `charly deploy add <name> <ref> --disposable` or mark a VM under `vm:` in `vm.yml`.

**After committing the source-level fix, `charly update` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
