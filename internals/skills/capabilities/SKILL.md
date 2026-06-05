---
name: capabilities
description: |
  MUST be invoked before any work involving: OCI label contract, Capabilities / BoxMetadata struct, CapabilityLabelMap completeness check, LabelService structured round-trip, source-less deploy via `ov deploy from-box`, or adding a new OCI label. Developer-facing; users author via `/ov-image:layer` and `/ov-image:image`.
---

# Capabilities — the OCI-label runtime contract

Every overthink image carries a complete snapshot of "what can this image do, what does it need, what does it provide" as `org.overthinkos.*` OCI labels. This skill documents the contract, the single Go type behind it, the completeness test that keeps it honest, and the source-less deploy path it enables.

Source of truth: `ov/capabilities.go` + `ov/labels.go`. See also `/ov-internals:go` for the broader Go architecture and `/ov-internals:install-plan` for how this feeds the build/deploy IR.

## The `Capabilities` type alias

```go
// ov/capabilities.go:29
type Capabilities = BoxMetadata
```

`Capabilities` is a **Go type alias** — not a separate struct. Every existing consumer that holds an `*BoxMetadata` (there are many) transparently participates in the capabilities contract. New code (K8s generator, `ov deploy from-box`) uses the canonical `Capabilities` name for readability. Aliasing (rather than wrapping) means there's exactly one place fields are defined: `BoxMetadata` in `ov/labels.go`.

Why this matters: adding a field anywhere in `BoxMetadata` automatically becomes part of the capabilities contract, which means it MUST have a `CapabilityLabelMap` entry. Forget the entry and CI blocks the PR.

## `CapabilityLabelMap` — field → OCI-label name

`ov/capabilities.go:35` names every label that participates in the contract. Entry grouping (identity / account / ports / security / networking / env / engine+init / distro+builder / hooks+vm / skills / data / dependency-graph / tests) mirrors the `BoxMetadata` field ordering so the map reads as a spec of the on-disk format.

Key entries:

| Field | Label const | What it stores |
|---|---|---|
| `Version` | `LabelVersion` (`org.overthinkos.version`) | The image's content-derived **`EffectiveVersion`** — its dedicated `version:` if set, else the highest layer `version:` across the chain (`ov/effective_version.go`). NOT the per-build tag; stable when no layer changed. Short-name resolution + `ov clean` retention prefer this label over the tag. Also the "is this an ov image?" presence sentinel (`ExtractMetadata` returns nil when empty). |
| `Service` | `LabelService` (`org.overthinkos.service`) | **Structured JSON array of `CapabilityService`** — not just names. 22 per-entry fields including `kind`, `events`, `auto_start`, `start_retries`, `priority`, `init`, `layer`. See "LabelService" below. |
| `Init` | `LabelInit` | Init system name (supervisord / systemd / none). |
| `ServiceNames` | `LabelInit` | Per-init active-name list; baked alongside `LabelInit` for CLI ergonomics (e.g., `ov service status`). |
| `Tests` | `LabelEval` | Three-section `{layer, image, deploy}` JSON — the tests baked into the image, consumed by `ov eval live` / `ov eval box`. See `/ov-eval:eval`. |
| `Shell` | `LabelShell` | Three-section `{layer, image, deploy}` JSON shell-init manifest. Each entry carries an Origin (layer name / "image" / "deploy"), an ID for overlay keying, an optional Generic body (intrinsic init + path_append) and a per-shell ByShell map (bash/zsh/fish/sh sub-blocks). Consumed by `ov box inspect`, `ov deploy from-box`, and `MergeDeployShell` for deploy.yml `shell:` overlay merging. See `/ov-image:layer` "Shell Init Surface". |
| `EnvProvide` / `MCPProvide` | `LabelEnvProvide` / `LabelMCPProvide` | Cross-container discovery: what env vars / MCP servers this image advertises to pod peers. |
| `EnvRequire` / `MCPRequire` | `LabelEnvRequire` / `LabelMCPRequire` | What this image *needs* from peers — validated at `ov config` time. |

## `TestCapabilityLabelCompleteness` — the guardrail

`ov/capabilities_test.go:TestCapabilityLabelCompleteness` runs on every `go test ./...` invocation. It uses `reflect.TypeOf(BoxMetadata{})` to enumerate every exported field and fails if any field is missing from `CapabilityLabelMap`:

```go
// ov/capabilities.go:116
func checkCapabilityLabelCompleteness() error {
    rt := reflect.TypeOf(BoxMetadata{})
    var missing []string
    for i := 0; i < rt.NumField(); i++ {
        name := rt.Field(i).Name
        if _, ok := CapabilityLabelMap[name]; !ok {
            missing = append(missing, name)
        }
    }
    // ...
}
```

This is the enforcement mechanism that keeps the OCI-label contract and the Go struct in sync. **Workflow for adding a capability:**

1. Add the field to `BoxMetadata` in `ov/labels.go` with a JSON tag.
2. Add the label const (`LabelFoo = "org.overthinkos.foo"`) next to the other label consts.
3. Add the `CapabilityLabelMap` entry: `"Foo": LabelFoo`.
4. Emit + parse the label in `EmitLabels` / `ExtractMetadata`.
5. `go test ./...` passes.

Skip step 3 and the test fails with `BoxMetadata fields without CapabilityLabelMap entry: [Foo]`.

## `LabelService` — structured per-entry service data

The services label is a full structured round-trip — not a flat list of names:

```go
// ov/labels.go (CapabilityService struct, paraphrased)
type CapabilityService struct {
    Name             string
    Scope            string            // system / user
    Enable           bool
    UsePackaged      string            // name of distro-shipped unit to reuse
    Exec             string
    Env              map[string]string
    Restart          string
    WorkingDirectory string
    User             string
    After            []string
    Before           []string
    Stdout           string
    StopTimeout      string
    Kind             string            // "program" (default) | "eventlistener"
    Events           string            // required when Kind == "eventlistener"
    AutoStart        *bool             // three-state; supervisord autostart=
    StartRetries     int
    StartSecs        int
    StopSignal       string
    ExitCodes        string
    Priority         int
    Init             string            // which init owns it (supervisord / systemd)
    Layer            string            // source layer name
}
```

**Why this matters**: `ov deploy from-box` (see below) reconstructs the full deploy surface from OCI labels alone. A names-only `Service` label would leave deploy-time K8s manifest generation blind to whether a process needs `start_retries: 3` or is an eventlistener. The structured label carries every supervisord directive faithfully, and the K8s Kustomize generator (see `/ov-kubernetes:kubernetes`) reads from it without touching the source repo.

## Source-less deploy: `ov deploy from-box`

`CapabilitiesFromLabels(engine, imageRef)` at `ov/capabilities.go:136` is the source-less entry point: given an engine + image ref, it runs `ExtractMetadata` (which pulls labels via `podman inspect` / `docker inspect`), returns a fully-populated `*Capabilities`, and every downstream consumer (deploy target, K8s generator, quadlet generator) reads from that struct.

```go
caps, err := CapabilitiesFromLabels("podman", "ghcr.io/overthinkos/fedora-coder:latest")
// caps.Service[0].Kind == "eventlistener" works — no source repo needed
```

This is what enables the "**K8s deploy without access to overthink.yml**" invariant: a Kustomize overlay can be generated from a published image alone, for dev/staging/prod clusters that never see the build repo.

## Three-layer image architecture (planned schema split)

The long-term direction (documented in `ov/capabilities.go:17-22`) is to split `BoxConfig` into three discriminated sections:

- `image.build:` — Containerfile inputs (base image, layers, distro/builder selection). Consumed only by `ov box build`.
- `image.capabilities:` — the runtime contract documented here. Emitted as OCI labels.
- `image.deploy:` — target-specific defaults (K8s storage class, container-target port defaults). Consumed by `ov deploy add`.

Today these co-exist in a single `BoxConfig`. The `Capabilities` alias is the stepping stone — once the schema split lands, `Capabilities` will point at the `capabilities:` subsection directly instead of aliasing the whole struct. The `CapabilityLabelMap` completeness test will keep the contract honest across the migration.

See `/ov-image:image` for current user-facing structure and `/ov-build:migrate` for the one-shot `ov migrate` converter that emits the new schema when the split lands.

## Adding a new OCI label — checklist

1. Add the const to `ov/labels.go` (`LabelFoo = "org.overthinkos.foo"`).
2. Add the field to `BoxMetadata` with a matching JSON tag.
3. Add the `CapabilityLabelMap` entry: `"Foo": LabelFoo`.
4. Populate it in `EmitLabels` (label emission at build time).
5. Parse it in `ExtractMetadata` (label read-back at deploy time).
6. If the field is a struct/list, prefer `mustMarshalJSON` for consistent encoding (see how `LabelService`, `LabelEval`, `LabelVolume` do it).
7. `go test ./...` — `TestCapabilityLabelCompleteness` passes.
8. Update `/ov-internals:capabilities` (this skill) with the new entry in the key-labels table.

## See Also

- `/ov-image:image` — user-facing `image:` entries in `overthink.yml`
- `/ov-image:layer` — user-facing `layer:` authoring, including `service:` which feeds `LabelService`
- `/ov-core:deploy` — `ov deploy add` / `from-image` / `sync` commands
- `/ov-kubernetes:kubernetes` — K8s deploy target that reads `LabelService` to generate Kustomize
- `/ov-eval:eval` — three-section `LabelEval` (layer/image/deploy) — same label-contract pattern
- `/ov-build:migrate` — `ov migrate` — emits the schema that populates these labels
- `/ov-internals:go` — Go architecture overview, `LoadUnified`, `parseLayerYAML`
- `/ov-internals:install-plan` — internal IR shared across build and deploy pipelines
