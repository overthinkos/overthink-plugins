---
name: plugin
description: |
  Use when authoring or modifying a charly PLUGIN — a candy with a `plugin:` block that contributes
  Providers (verbs/kinds/deploy-targets/steps/builders/commands), its own CUE schema, builtin (compiled-in) or
  external (out-of-tree git repo). Covers the unified Provider model, the per-plugin CUE-schema contract
  (single source → Go params for dev + schema-over-Describe RPC for runtime), the SDK, and the loader.
---

# Plugins — the unified Provider + per-plugin CUE-schema model

A **plugin is a candy** (`candy/<name>/charly.yml`) carrying a `plugin:` block. The single addition that
makes a candy a plugin is that block:

```yaml
my-plugin:
  candy:
    version: 2026.180.1200          # mandatory CalVer (any candy)
    description: |-                 # mandatory (ADE)
      What this plugin provides.
  my-plugin-decl:
    plugin:
      providers: [verb:myprobe]     # "<class>:<word>" — class ∈ kind|verb|deploy|step|builder|command
      source: builtin               # OR github.com/org/repo/candy/<name>  (out-of-tree)
  myprobe-check:                     # ADE: ≥1 deterministic check (its OWN acceptance test)
    check: the myprobe verb dispatches and passes
    plugin: myprobe
    plugin_input: { marker: hello }
    context: [runtime]
```

A candy with no `plugin:` block is an ordinary candy; one WITH it is a plugin. Full candy authoring surface
applies (`/charly-image:layer`), including the mandatory `version:`/`description:`/`plan:`+`check:` (ADE).

## One Provider, transport-invisible

Every reserved word — every kind, verb, deploy-target, step, builder, command — is served by ONE `Provider`
(`charly/provider.go`): `Reserved() string`, `Class() ProviderClass`, `Invoke(ctx, *Operation) (*Result,
error)`. A Provider is IN-PROCESS (a builtin, registered from `init()`) or OUT-OF-PROCESS (an external,
served over go-plugin gRPC). The registry (`providerRegistry`, `provider_registry.go`), the call sites, and
the bijection gate treat both identically — **the transport is invisible above the registry**. A
`check: { plugin: <word>, plugin_input: {…} }` step dispatches through `runPluginVerb` →
`providerRegistry.ResolveVerb(word)` → `Invoke`, whether the provider is compiled in or out-of-process.

## Placement: builtin OR external, in-proc OR out-of-process — at deploy AND build time

Placement is a free, per-plugin choice, invisible above the registry: a provider runs EITHER compiled into
`charly` (a builtin, in-process) OR dynamically loaded from an out-of-tree candy (an external, out-of-process
over go-plugin gRPC), and the SAME provider works in either placement with ZERO authoring change. The strategic
direction is that every internal/builtin provider migrates external over time — so author every plugin
placement-agnostic from day one, and design it to work either way.

**Every class is placement-free** — kind, verb, deploy-target, step, builder, AND command are ALL
external-capable. An external (out-of-tree) COMMAND plugin contributes a `charly <word>` CLI subcommand
dispatched OUT-OF-PROCESS: the declared word is prescanned into the Kong grammar before parse, and the provider
is LAZY-connected on the first actual `charly <word>` invocation, which forwards the pass-through args via
`Invoke(OpRun)`. `builtinCommandBase.Invoke` returning "in-process only" is the BUILTIN command path ALONE (a
builtin command contributes via its static `KongCommand()` + Go `Run` handler and never serves itself
out-of-process) — NOT a class-level limit. Each class's leg differs by lifecycle phase: verb/deploy/step/
builder ride build (`OpEmit`/`OpResolve`) and/or deploy (`OpExecute`); command rides CLI invocation (`OpRun`);
a **kind** rides CONFIG LOAD (`OpLoad`) — its serving plugin is recognized + connected at config-PARSE by the
F4 prescan (`registerDeclaredKind` + `connectDeclaredKindPlugins`, re-entrancy-guarded), so a `kind: <word>`
entity whose plugin is NOT compiled in decodes via `runPluginKind` during load. Reference (out-of-process-only):
`candy/plugin-example-kind`; loader mechanism: `/charly-internals:go` (`plugin_prescan.go`).

**A kind decode is FLAT or STRUCTURAL (F5).** A FLAT kind (the default) lands its `OpLoad` body OPAQUELY in
`uf.PluginKinds[disc][name]` (F4). A STRUCTURAL kind sets `ProvidedCapability.Structural = true` (the proto
`structural` field) in its Describe — its `OpLoad` returns a `spec.Deploy` (BundleNode) MEMBER TREE the host
folds into `uf.Bundle`, the SAME map a builtin pod/group/candy decoder populates, so the entity participates in
deploy/check exactly like a builtin (the folded member goes through the SAME `validateDeploy`). This is the
channel that lets the seven builtin structural kind decoders (pod/vm/k8s/local/android/group/candy) be
EXTERNALIZED. Reference (out-of-process-only): `candy/plugin-example-structkind`; the host fold is `runPluginKind`
(`/charly-internals:go`).
See "Authoring an external COMMAND plugin" below.

- **The perf invariant that makes placement free.** A builtin dispatches through its typed in-proc fast path
  (`CheckVerbProvider.RunVerb` / `KindProvider.DecodeNode` / `DeployTargetProvider.ResolveTarget` /
  `StepProvider.Emit*`) and NEVER marshals the Op into the serializable `Invoke`
  envelope; the JSON envelope (`marshalJSON`) is paid ONLY out-of-process. Choosing builtin therefore costs no
  envelope tax — placement is a free build/deploy decision, not a performance trade-off. Locked by
  `TestPerfGate_BuiltinVerbsSkipEnvelope` + `BenchmarkVerbTypedDispatchFork` (0-alloc) vs
  `BenchmarkVerbEnvelopeMarshal` (`provider_bench_test.go`).
- **Deploy time.** An external deploy-target provider runs its full Add/Test/Update/Del lifecycle over the
  host-served executor reverse channel — the plugin applies the deployment's ops on the real venue it cannot
  hold across the process boundary (`OpExecute`), and the host records the returned teardown ops to the ledger.
  A bed/deploy that uses an external deploy SUBSTRATE word is recognized at config-PARSE time (before the
  provider connects) and routed host-side by the shared check classifier. **A substrate may ALSO bring its OWN
  host-side venue LIFECYCLE + PRERESOLVE (F6):** a `class:deploy` capability declaring `Lifecycle=true` gets a
  wire-backed `substrateLifecycle` registered at plugin-load — the host calls its `OpPrepareVenue`/`OpStart`/
  `OpStop`/`OpStatus`/`OpRebuild`/… (host→plugin on `Provider.Invoke`), and `OpPrepareVenue` returns a
  `spec.VenueDescriptor` the host re-materializes into a real executor (the live executor never crosses the
  wire). One declaring `Preresolve=true` gets a wire-backed `deployPreresolver` (`OpPreresolve` → the opaque
  `DeployVenue.Substrate` payload), generalizing the in-core k8s/android preresolvers. Reference
  (out-of-process-only): `candy/plugin-example-lifecycle`; mechanism: `/charly-internals:install-plan`
  (`substrate_lifecycle_grpc.go`). This is the channel M4 reuses to externalize the pod/vm lifecycles. An external **`run:` plugin verb /
  step** composed INSIDE a deploy (a `local:`/`vm:` target, where the install runs ON the target, not baked
  into an image) likewise EXECUTES at deploy: it lowers to an `ExternalPluginStep` IR node which the external
  `local:`/`vm:` deploy walk reaches as a host-engine step over `RunHostStep`, where `executeExternalPluginStep`
  `Invoke(OpExecute)`s WITH the live `DeployExecutor` on the SAME reverse channel, so the plugin runs
  its deploy-context effect on the target and RETURNS its teardown `ReverseOp`s, which the host records to
  the ledger and replays at `charly bundle del` (record-and-replay, the SAME `spec.DeployReply` wire as the
  deploy target — R3). Only an EXTERNAL provider is routed there (the `executorInvoker` discriminator,
  satisfied SOLELY by the out-of-process `grpcProvider`); a builtin `ProvisionActor` verb keeps its in-proc
  shell path. So the verb/step class is external-capable at BOTH build (`OpEmit`, next bullet) AND deploy
  (`OpExecute`), placement-agnostic. Detail → `/charly-internals:install-plan` (the `externalDeployTarget`
  lifecycle + the `ExternalPluginStep` IR kind + the `OpExecute` reverse channel).
- **Build time.** `charly box build` / `charly box generate` connect the project's external plugin candies during
  image generation, so a plugin EXECUTES at build to emit its Containerfile contribution, placement-agnostically
  (a builtin in-proc, an external over gRPC) — and BOTH the verb/step leg AND the builder leg ride the SAME
  connect seam, class-agnostically:
  - a `run:` plugin **verb / step** returns a Containerfile fragment via `OpEmit` → `spec.EmitReply.Fragment`,
    spliced verbatim into the Containerfile (egress-validated).
  - an external **builder** (`ClassBuilder`) selected by a candy's `external_builder: <word>` field returns a
    multi-stage build via `Invoke(OpResolve)` → `spec.BuilderResolveReply`: its `Stage` (a `FROM <ref> AS <name>`
    block) is spliced PRE-main-FROM and its `CopyArtifacts` (`COPY --from=<stage> …`) POST-main-FROM. This is the
    build-time BUILDER leg — the multi-stage counterpart of the verb/step OpEmit leg, so `builder` is an
    external-capable class at build too (alongside verb/kind/deploy/step). The `command` class has no build-time
    leg — a command dispatches at CLI invocation, not at build — and is external-capable THERE via `Invoke(OpRun)`
    (the Placement paragraph above), so EVERY class (kind/verb/deploy/step/builder/command) is external-capable.
  This is operator-authorized build-time execution of host-built plugin code: a project's composed external
  plugins run as host code during its image builds. Detail → `/charly-build:generate` + `/charly-internals:generate-source`.

## The per-plugin CUE schema — the single source, two consumers

**Every plugin ships its OWN `.cue` schema, and it is the SINGLE SOURCE for that plugin's params.** It is
SELF-CONTAINED (package-less, references no base def) and used two ways — the SAME contract core `spec` uses:

1. **DEV-TIME → Go params.** `cue exp gengotypes` (driven by `task cue:gen`, which wraps the schema with
   `package params` + `@go(params)`) emits the plugin's `params/cue_types_gen.go`. The provider decodes
   `plugin_input` into that TYPED struct — never a hand-parsed `map[string]any`, never a hand-written struct.
2. **RUNTIME → schema-over-RPC.** The plugin SERVES its `.cue` source over the Provider **`Describe`**
   channel (the proto `Capabilities.schema_cue` field + structured `ProvidedCapability{class,word,input_def}`).
   The host splices it onto charly's base schema (`base ++ plugin`, via `internal/schemaconcat` — the SAME
   concat contract as the runtime `sharedCueSchema`, R3) and validates every authored `plugin_input` against
   the plugin's def (e.g. `#MyprobeInput`). The host **never reads a candy's `schema/` dir from disk** — the
   schema travels WITH the plugin.

**A `class:step` plugin ALSO declares its install-step contract over Describe (F3).** A step plugin (a
PLUGIN-contributed install-step KIND, distinct from a `class:verb` step which rides the fixed
`ExternalPlugin` kind) sets `ProvidedCapability.StepContract{Scope,Venue,Gate}` (the proto
`step_contract` field; `sdk.ProvidedCapability.StepContract`) — the host carries that DECLARED contract so
a `run: plugin: <word>` lowers to an `externalStep` (kind `external:<word>`, opaque Payload) the OPEN
DEFAULT ARM dispatches via `OpExecute`, with NO compiled-in case. Reverse is NOT declared (an external
step's teardown ops are recorded dynamically from its `OpExecute` reply). Authoring + IR mechanism:
`/charly-internals:install-plan` (the `externalStep` row); reference: `candy/plugin-example-stepkind`.

**Zero builtin/external distinction in schema handling.** Both arrive at the host as a `PluginUnit`
(`Providers` + `Schema`) from `PluginTransport.Connect` — `InProcTransport` for a builtin, `LocalTransport`
(go-plugin exec) for an external. The host's load/gate/validate code NEVER branches on which kind it is. The
only difference is WHERE the plugin runs (compiled into charly vs a separate process), and therefore WHEN its
providers register (a builtin at `init()`, an external at deploy/check load) — a load-TIMING property, never a
schema-handling one.

### The load gate + the validator (one each, shared)

- `registerPluginUnitSchema(name, schema)` is THE load gate (`plugin_loader.go`), byte-identical for builtin
  and external: it rejects an **empty** schema, a schema that will not **splice** onto the base, and a declared
  `input_def` the schema does not define — all LOUD failures at load. Builtins are gated at process start
  (`loadBuiltinPluginUnits`, a `sync.Once` pass over every registered unit); externals at connect
  (`loadPluginUnit` → build on host → `LocalTransport.Connect` → gate → register).
- `validateAuthoredPluginInput(class, word, json)` is THE validator: it looks the def up in the process-wide
  `pluginSchemas` set (filled by the gate) and unifies the authored input against it. Wired into
  `runPluginVerb` before dispatch — a missing/empty/typo'd field is a hard `TestFail`, not a silent surprise.

## Authoring a BUILTIN plugin (Go in charly's own module)

A LEGACY builtin's Go lives in charly's module itself (so it needs no `go.work` entry — this is
the in-charly-module path, distinct from a candy COMPILED IN via `compiled_plugins:`, which DOES
ride `go.work`; see "Authoring a COMPILED-IN plugin candy" below). This path is being RETIRED as
each builtin relocates into a candy; the remaining in-charly-module units (`examplerunverb` + the
host-coupled check verbs/kinds that keep the live `*Runner`/`*Generator`) await the engine-kit
extraction. Mirror `charly/plugin/builtins/examplerunverb/`:

- `schema/<name>.cue` — the self-contained `#<Word>Input` def (e.g. `{ marker: string & !="" }`).
- `params/cue_types_gen.go` — generated by `task cue:gen` (it loops `charly/plugin/builtins/*/schema`).
- `plugin.go` (`package <name>`) — `//go:embed schema/*.cue` + `Schema()` (concat via `schemaconcat`) +
  `var InputDefs = map[string]string{"verb:<word>": "#<Word>Input"}`.
- The Provider + the unit registration live in `package main` (e.g. `plugin_examplerunverb.go`):
  `func init() { RegisterBuiltinPluginUnit(PluginUnit{Providers: []Provider{…}, Schema: PluginSchema{CueSource:
  <pkg>.Schema(), InputDefs: <pkg>.InputDefs}}) }`. An in-charly-module builtin registers its providers
  directly at `init()` and needs NO candy manifest — it is not selected via `compiled_plugins:` (the
  retired `source: builtin` candy-linkage form is gone; the last one, exampleprobe, is now a real
  candy module — see below).

## Authoring an EXTERNAL plugin (out-of-tree git repo)

The candy IS its own Go module (`go.mod` + `replace …/charly => …` while in-repo). Mirror
`candy/plugin-example-external/`:

- `schema/<name>.cue` — the self-contained def (same shape as a builtin's).
- `params/cue_types_gen.go` — generated the same way (the gen loop also covers the in-repo example).
- `main.go` — `func main() { sdk.Serve(&provider{}, &meta{}) }`; the provider decodes `plugin_input` into the
  generated `params.<Word>Input`; `meta.Describe` returns `sdk.BuildCapabilities(calver, []sdk.ProvidedCapability{…},
  schemaFS, "schema")` (it `//go:embed`s `schema/*.cue`). `BuildCapabilities` concatenates + compiles the
  schema STANDALONE (failing loudly before serving) and ships the source in `schema_cue`.
- The candy's `plugin.source` is the `github.com/org/repo/candy/<name>` ref. charly's loader fetches the repo
  (the same `@github` resolver candies use), `go build`s the provider binary ON THE HOST, and connects
  out-of-process via `LocalTransport`. The host build runs with `GOWORK=off` so a repo-root `go.work` cannot
  reject a non-member candy dir — the out-of-process build is always standalone in the candy's own module.

## Authoring a COMPILED-IN plugin candy (the candy compiled INTO charly)

The SAME out-of-tree candy can be COMPILED INTO charly — the in-proc placement of a candy, selected
per-charly-build by `charly.yml` `compiled_plugins:`. This ships a plugin candy INSIDE the binary
WITHOUT its Go living in charly's module (it rides `go.work`). `candy/plugin-example-external/` is the
reference: it is BOTH the out-of-process example above AND the compiled-in example — one provider, two
placements, ZERO authoring change.

- The provider lives in the candy's IMPORTABLE root package (`NewProvider() pb.ProviderServer` +
  `NewMeta() pb.PluginMetaServer` + `//go:embed schema/*.cue`), NOT `package main`. `main.go` moves to
  `cmd/serve/` as a 3-line `sdk.Serve(<pkg>.NewProvider(), <pkg>.NewMeta())` shim (the out-of-process
  entrypoint, host-built only when the candy is NOT compiled in).
- List the candy in `compiled_plugins:` (the embedded `charly/charly.yml` for the default binary; a
  consumer's own `charly.yml` for a custom footprint — "which plugins are in the binary" is a normal
  candy-inclusion choice).
- `pluginsgen` (`charly/internal/pluginsgen`, run by `task build:charly` before `go build`, `GOWORK=off`,
  stdlib+yaml only) reads `compiled_plugins:` and emits `charly/plugins_generated.go` (one
  `registerCompiledPlugin(<pkg>.NewProvider(), <pkg>.NewMeta())` per candy) + the repo-root `go.work` (a
  `use ./candy/<name>` per candy, so `go build ./charly` resolves the candy imports). Both are COMMITTED
  + reproducibility-gated (`TestPluginsGenReproducible`).
- `registerCompiledPlugin` drives `InProcServedTransport`: it calls the candy's `Describe` IN-PROCESS, runs
  the SAME `buildUnit` capability-lift + schema gate an external goes through (so the compiled-in schema
  enters the SAME `loadBuiltinPluginUnits` gate), and registers each capability wrapped in an
  `inprocProvider` (origin `"builtin"`) — the in-proc twin of `grpcProvider`. Dispatch is registry-routed,
  transport-invisible.
- COEXIST: a candy compiled in (origin `"builtin"`) is SKIPPED by `loadProjectPlugins` (the out-of-process
  host build is redundant); a candy NOT in `compiled_plugins:` still host-builds + connects out-of-process
  when a plan references its word. Placement is a per-charly-build choice, invisible above the registry.
- PERF: a compiled-in CANDY dispatches through the pb `Invoke` envelope IN-PROCESS (no socket) via
  `inprocProvider`, whereas a LEGACY in-charly-module builtin uses its typed fast path
  (`RunVerb`/`DecodeNode`, no envelope). The candy path pays the JSON envelope but not the gRPC transport.

## Authoring a HOST-COUPLED check-verb candy (the kit — dual-placement via the reverse channel)

A check verb whose logic needs the LIVE check engine — exec-in-container, host-vantage HTTP, host TCP
dial (`file`/`http`/`port`/`command`/`service`/…) — implements **`charly/plugin/kit`**, the importable
contract for the check engine. It runs in EITHER placement, invisibly above the registry: COMPILED-IN
(charly passes the live `*Runner` as the `kit.CheckContext`) OR OUT-OF-PROCESS (the CheckContext legs are
served back over the host's reverse channel — ExecutorService for `cc.Exec()` + CheckContextService for
`cc.HTTPDo`/`cc.AddBackground`, F2 — while the scalar legs `Mode`/`Box`/`Instance`/`Distros`/`DialTimeout`
ride the env_json snapshot). `candy/plugin-port` + `candy/plugin-http` are the reference (both served
OUT-OF-PROCESS in the default build); the remaining kit candies are compiled-in by default until M1 adds
their `cmd/serve` shim. Shape:

- The candy's importable root package implements `kit.CheckVerbProvider` (`Reserved()` + `RunVerb(ctx,
  cc kit.CheckContext, op *spec.Op) kit.Result`) — the verb's logic (formerly the `r.runX` *Runner
  method) lives HERE, reaching the deployment through `cc.Exec().RunCapture` / `cc.Mode()` /
  `cc.DialTimeout()` / `cc.HTTPDo(...)` / `cc.Distros()`. It exports `NewCheckVerb() kit.CheckVerbProvider`
  + the raw schema `embed.FS` (`SchemaFS` + `SchemaDir`) + `InputDefs`.
- COMPILED-IN: charly's `registerCompiledCheckVerb` (generated into `plugins_generated.go` by pluginsgen,
  which detects the kit shape by the exported `NewCheckVerb`) wraps it in a `kitVerbAdapter` (a package-main
  `CheckVerbProvider` that passes the live `*Runner` as a `kit.CheckContext` via `runnerCheckContext` and
  converts `kit.Result`→`CheckResult`), concatenates the candy's schema (the candy can't import
  `internal/schemaconcat`), and registers through the SAME `RegisterBuiltinPluginUnit` gate (origin
  `"builtin"`). Dispatch is the SAME `runOne`→`CheckVerbProvider.RunVerb` path an in-charly-module verb
  uses — full typed fast path, the real `*Runner`, no envelope.
- OUT-OF-PROCESS: a `cmd/serve/main.go` shim calls `sdk.ServeCheckVerb(pkg.NewCheckVerb(), calver,
  pkg.SchemaFS, pkg.SchemaDir, pkg.InputDefs)`, which wraps the kit verb in a pb.ProviderServer whose Invoke
  reconstructs a `sdkCheckContext` from the InvokeRequest broker (ExecutorService + CheckContextService) +
  the env_json snapshot, then runs `RunVerb`. The host go-builds `./cmd/serve` + connects via
  `LocalTransport` when the candy is NOT in `compiled_plugins`; `invokeVerbProvider` (provider_checkenv.go)
  serves BOTH reverse services on one broker id. The verdict round-trips as the same `{status,message}`
  every out-of-process verb returns.
- `kit` imports only the stdlib + `charly/spec`; the candy imports `kit` + `sdk` + `spec` + its `params`.

The progression: a LEGACY in-charly-module check verb (RunVerb on `*Runner`) → a kit candy (RunVerb on
`kit.CheckContext`), relocating the verb's Go out of charly's module — runnable in-proc (the typed fast
path) OR out-of-process (the reverse channel) with ZERO authoring change.

The SDK (`charly/plugin/sdk`) is the ONLY charly package an external module imports — `Serve`,
`ServeCheckVerb` (the kit-verb out-of-process serve entry), `Handshake`, `BuildCapabilities`,
`ProvidedCapability`, `Conn`. `schemaconcat` stays `internal/` (the SDK, under `charly/`,
imports it; the external module reaches it only transitively through the SDK).

## Authoring an external COMMAND plugin (a `charly <word>` subcommand)

A `command:<word>` provider is an external plugin (authored exactly like the verb-class one above — its own Go
module, self-contained `schema/<name>.cue`, generated `params/`, `sdk.Serve` + `BuildCapabilities`) that
contributes a TOP-LEVEL `charly <word>` CLI subcommand. Mirror `candy/plugin-example-command/`:

- `plugin.providers: [command:<word>]` + `source: github.com/org/repo/candy/<name>` in the candy's `plugin:`
  block; `Describe` returns the capability with `Class: "command", Word: "<word>"`.
- `Invoke` handles `OpRun` (the command-run selector): the host forwards the user's pass-through CLI tokens
  from `charly <word> <args…>` as `op.Params = {"args": [...]}` marshalled DIRECTLY — NOT wrapped in the
  `plugin_input` envelope a verb CHECK step uses. The provider decodes that into its generated typed params
  struct (e.g. `params.<Word>Input` with an `Args []string` field) and does its effect.

The discovery → grammar → dispatch flow (host side, owned by `charly/plugin_command_prescan.go` +
`provider_command_external.go`): when the candy is discovered in the project, a byte-gated prescan
(`prescanProjectCommandWords`, run in `main` BEFORE `kong.Parse`) registers the declared word so `charly <word>`
PARSES; `collectExternalCommandPlugins` builds a Kong grammar holder for it with the provider UNconnected. The
host build + gRPC connect is LAZY — paid ONLY when the user actually runs the command: `dispatchExternalCommand`
calls `connectCommandPlugin` (scoped to the one word) then `Invoke(OpRun, {"args":[…]})`. So every `charly`
invocation that is NOT `charly <word>` is byte-for-byte unaffected. (A builtin command, by contrast, contributes
via its compiled-in `CommandProvider.KongCommand()` + Go `Run` handler — `provider_command.go`.)

## Why self-contained schemas

A plugin's `#<Word>Input` references NO base def, so it compiles STANDALONE — the exact property
`cue exp gengotypes` needs to generate Go params, AND the property that lets the SDK compile it serve-side.
The host's `base ++ plugin` splice therefore exists to detect a def-name **collision** with the base, not to
resolve base references. (`TestExternalSchemaSelfContained` proves a base-referencing schema fails a standalone
compile.)

## Verification

- `go test ./...` — the registry/transport/schema seams (`TestPluginGRPCRoundTrip`,
  `TestExternalPluginEndToEnd` proves the schema travels over RPC, `TestPluginSchemaSpliceValidation`,
  `TestBuiltinPluginSchemasSplice` is the CI gate that every builtin schema splices).
- `task cue:gen` — regenerates spec + every plugin's params; reproducible (a second run is a no-op).
- `charly box validate` — the candy + `plugin:` block (`validatePluginCandy` checks each capability is
  well-formed and, for `source: builtin`, that the provider is actually compiled in).
- R10: a disposable bed composing a builtin AND an external plugin, driven to a fresh `charly update` — the
  builtin's baked check + the external's out-of-process check both pass.

## Cross-References

- `/charly-internals:go` — the provider registry + the reserved-word spine (verbs/kinds/deploy/steps/builders/commands).
- `/charly-image:layer` — the candy authoring surface the `plugin:` block extends.
- `/charly-check:check` — the `plugin:` check verb + ADE (a plugin's own acceptance plan).
- `/charly-build:validate` — `charly box validate` rules.
- `/charly-internals:install-plan` — the `externalDeployTarget` deploy lifecycle (`OpExecute` reverse channel, ledger record) + the `OpEmit` build-time fragment; the deploy wire types in `charly/spec/deploy_wire.go`.
- `/charly-build:generate` + `/charly-internals:generate-source` — the build-time plugin connect seam + the `emitTasks` placement-agnostic plugin-verb dispatch (`OpEmit` → fragment).

## When to Use This Skill

Invoke before authoring or editing any `plugin:` block, any `charly/plugin/**` code, the plugin SDK, a builtin
unit under `charly/plugin/builtins/`, an external plugin module, or the plugin schema/param gen pipeline.
