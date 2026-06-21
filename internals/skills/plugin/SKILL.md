---
name: plugin
description: |
  Use when authoring or modifying a charly PLUGIN — a candy with a `plugin:` block that contributes
  Providers (verbs/kinds/deploy-targets/steps/builders), its own CUE schema, builtin (compiled-in) or
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
      providers: [verb:myprobe]     # "<class>:<word>" — class ∈ kind|verb|deploy|step|builder
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

Every reserved word — every kind, verb, deploy-target, step, builder — is served by ONE `Provider`
(`charly/provider.go`): `Reserved() string`, `Class() ProviderClass`, `Invoke(ctx, *Operation) (*Result,
error)`. A Provider is IN-PROCESS (a builtin, registered from `init()`) or OUT-OF-PROCESS (an external,
served over go-plugin gRPC). The registry (`providerRegistry`, `provider_registry.go`), the call sites, and
the bijection gate treat both identically — **the transport is invisible above the registry**. A
`check: { plugin: <word>, plugin_input: {…} }` step dispatches through `runPluginVerb` →
`providerRegistry.ResolveVerb(word)` → `Invoke`, whether the provider is compiled in or out-of-process.

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

## Authoring a BUILTIN plugin (compiled into charly)

A builtin's Go lives in charly's module (no `go.work` — charly cannot import `candy/`). Mirror
`charly/plugin/builtins/exampleprobe/`:

- `schema/<name>.cue` — the self-contained `#<Word>Input` def (e.g. `{ marker: string & !="" }`).
- `params/cue_types_gen.go` — generated by `task cue:gen` (it loops `charly/plugin/builtins/*/schema`).
- `plugin.go` (`package <name>`) — `//go:embed schema/*.cue` + `Schema()` (concat via `schemaconcat`) +
  `var InputDefs = map[string]string{"verb:<word>": "#<Word>Input"}`.
- The Provider + the unit registration live in `package main` (e.g. `plugin_example.go`):
  `func init() { RegisterBuiltinPluginUnit(PluginUnit{Providers: []Provider{…}, Schema: PluginSchema{CueSource:
  <pkg>.Schema(), InputDefs: <pkg>.InputDefs}}) }`. The candy (`candy/<name>/charly.yml`,
  `source: builtin`) links it by reserved word; no schema is read from the candy dir.

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
  out-of-process via `LocalTransport`.

The SDK (`charly/plugin/sdk`) is the ONLY charly package an external module imports — `Serve`, `Handshake`,
`BuildCapabilities`, `ProvidedCapability`, `Conn`. `schemaconcat` stays `internal/` (the SDK, under `charly/`,
imports it; the external module reaches it only transitively through the SDK).

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

- `/charly-internals:go` — the provider registry + the reserved-word spine (verbs/kinds/deploy/steps/builders).
- `/charly-image:layer` — the candy authoring surface the `plugin:` block extends.
- `/charly-check:check` — the `plugin:` check verb + ADE (a plugin's own acceptance plan).
- `/charly-build:validate` — `charly box validate` rules.

## When to Use This Skill

Invoke before authoring or editing any `plugin:` block, any `charly/plugin/**` code, the plugin SDK, a builtin
unit under `charly/plugin/builtins/`, an external plugin module, or the plugin schema/param gen pipeline.
