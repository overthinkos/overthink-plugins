---
name: capabilities
description: |
  MUST be invoked before any work involving: OCI label contract, Capabilities / ImageMetadata struct, CapabilityLabelMap completeness check, LabelServices structured round-trip, source-less deploy via `ov deploy from-image`, or adding a new OCI label. Developer-facing; users author via `/ov:layer` and `/ov:image`.
---

# Capabilities — the OCI-label runtime contract

Every overthink image carries a complete snapshot of "what can this image do, what does it need, what does it provide" as `org.overthinkos.*` OCI labels. This skill documents the contract, the single Go type behind it, the completeness test that keeps it honest, and the source-less deploy path it enables.

Source of truth: `ov/capabilities.go` + `ov/labels.go`. See also `/ov-dev:go` for the broader Go architecture and `/ov-dev:install-plan` for how this feeds the build/deploy IR.

## The `Capabilities` type alias

```go
// ov/capabilities.go:29
type Capabilities = ImageMetadata
```

`Capabilities` is a **Go type alias** — not a separate struct. Every existing consumer that holds an `*ImageMetadata` (there are many) transparently participates in the capabilities contract. New code (K8s generator, `ov deploy from-image`) uses the canonical `Capabilities` name for readability. Aliasing (rather than wrapping) means there's exactly one place fields are defined: `ImageMetadata` in `ov/labels.go`.

Why this matters: adding a field anywhere in `ImageMetadata` automatically becomes part of the capabilities contract, which means it MUST have a `CapabilityLabelMap` entry. Forget the entry and CI blocks the PR.

## `CapabilityLabelMap` — field → OCI-label name

`ov/capabilities.go:35` names every label that participates in the contract. Entry grouping (identity / account / ports / security / networking / env / engine+init / distro+builder / hooks+vm / skills / data / dependency-graph / tests) mirrors the `ImageMetadata` field ordering so the map reads as a spec of the on-disk format.

Key entries (post-services-cutover):

| Field | Label const | What it stores |
|---|---|---|
| `Services` | `LabelServices` (`org.overthinkos.services`) | **Structured JSON array of `CapabilityService`** — not just names. 22 per-entry fields including `kind`, `events`, `auto_start`, `start_retries`, `priority`, `init`, `layer`. See "LabelServices" below. |
| `Init` | `LabelInit` | Init system name (supervisord / systemd / none). |
| `ServiceNames` | `LabelInit` | Per-init active-name list; baked alongside `LabelInit` for CLI ergonomics (e.g., `ov service status`). |
| `Tests` | `LabelEval` | Three-section `{layer, image, deploy}` JSON — the tests baked into the image, consumed by `ov test` / `ov eval image`. See `/ov:eval`. |
| `EnvProvides` / `MCPProvides` | `LabelEnvProvides` / `LabelMCPProvides` | Cross-container discovery: what env vars / MCP servers this image advertises to pod peers. |
| `EnvRequires` / `MCPRequires` | `LabelEnvRequires` / `LabelMCPRequires` | What this image *needs* from peers — validated at `ov config` time. |

## `TestCapabilityLabelCompleteness` — the guardrail

`ov/capabilities_test.go:TestCapabilityLabelCompleteness` runs on every `go test ./...` invocation. It uses `reflect.TypeOf(ImageMetadata{})` to enumerate every exported field and fails if any field is missing from `CapabilityLabelMap`:

```go
// ov/capabilities.go:116
func checkCapabilityLabelCompleteness() error {
    rt := reflect.TypeOf(ImageMetadata{})
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

1. Add the field to `ImageMetadata` in `ov/labels.go` with a JSON tag.
2. Add the label const (`LabelFoo = "org.overthinkos.foo"`) next to the other label consts.
3. Add the `CapabilityLabelMap` entry: `"Foo": LabelFoo`.
4. Emit + parse the label in `EmitLabels` / `ExtractMetadata`.
5. `go test ./...` passes.

Skip step 3 and the test fails with `ImageMetadata fields without CapabilityLabelMap entry: [Foo]`.

## `LabelServices` — structured per-entry service data

Before the 2026-04 services-schema cutover, the services label was a flat list of names. After the cutover, it's a full structured round-trip:

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

**Why this matters**: `ov deploy from-image` (see below) reconstructs the full deploy surface from OCI labels alone. Pre-cutover, `Services` was just names — deploy-time K8s manifest generation couldn't know a process needed `start_retries: 3` or was an eventlistener. Now the label carries every supervisord directive faithfully, and the K8s Kustomize generator (see `/ov:kubernetes`) reads from it without touching the source repo.

## Source-less deploy: `ov deploy from-image`

`CapabilitiesFromLabels(engine, imageRef)` at `ov/capabilities.go:136` is the source-less entry point: given an engine + image ref, it runs `ExtractMetadata` (which pulls labels via `podman inspect` / `docker inspect`), returns a fully-populated `*Capabilities`, and every downstream consumer (deploy target, K8s generator, quadlet generator) reads from that struct.

```go
caps, err := CapabilitiesFromLabels("podman", "ghcr.io/overthinkos/fedora-coder:latest")
// caps.Services[0].Kind == "eventlistener" works — no source repo needed
```

This is what enables the "**K8s deploy without access to overthink.yml**" invariant: a Kustomize overlay can be generated from a published image alone, for dev/staging/prod clusters that never see the build repo.

## Three-layer image architecture (planned schema split)

The long-term direction (documented in `ov/capabilities.go:17-22`) is to split `ImageConfig` into three discriminated sections:

- `image.build:` — Containerfile inputs (base image, layers, distro/builder selection). Consumed only by `ov image build`.
- `image.capabilities:` — the runtime contract documented here. Emitted as OCI labels.
- `image.deployment:` — target-specific defaults (K8s storage class, container-target port defaults). Consumed by `ov deploy add`.

Today these co-exist in a single `ImageConfig`. The `Capabilities` alias is the stepping stone — once the schema split lands, `Capabilities` will point at the `capabilities:` subsection directly instead of aliasing the whole struct. The `CapabilityLabelMap` completeness test will keep the contract honest across the migration.

See `/ov:image` for current user-facing structure and `/ov:migrate` for the one-shot `ov migrate unified` converter that emits the new schema when the split lands.

## Adding a new OCI label — checklist

1. Add the const to `ov/labels.go` (`LabelFoo = "org.overthinkos.foo"`).
2. Add the field to `ImageMetadata` with a matching JSON tag.
3. Add the `CapabilityLabelMap` entry: `"Foo": LabelFoo`.
4. Populate it in `EmitLabels` (label emission at build time).
5. Parse it in `ExtractMetadata` (label read-back at deploy time).
6. If the field is a struct/list, prefer `mustMarshalJSON` for consistent encoding (see how `LabelServices`, `LabelEval`, `LabelVolumes` do it).
7. `go test ./...` — `TestCapabilityLabelCompleteness` passes.
8. Update `/ov-dev:capabilities` (this skill) with the new entry in the key-labels table.

## See Also

- `/ov:image` — user-facing `image:` entries in `overthink.yml`
- `/ov:layer` — user-facing `layer:` authoring, including `service:` which feeds `LabelServices`
- `/ov:deploy` — `ov deploy add` / `from-image` / `sync` commands
- `/ov:kubernetes` — K8s deploy target that reads `LabelServices` to generate Kustomize
- `/ov:eval` — three-section `LabelEval` (layer/image/deploy) — same label-contract pattern
- `/ov:migrate` — `ov migrate unified` — emits the schema that populates these labels
- `/ov-dev:go` — Go architecture overview, `LoadUnified`, `parseLayerYAML`
- `/ov-dev:install-plan` — internal IR shared across build and deploy pipelines
