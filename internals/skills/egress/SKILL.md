---
name: egress
description: |
  CUE EGRESS validation — validating (and, where it adds value, generating) the
  config files charly WRITES to a system BEFORE the bytes hit disk. MUST be
  invoked before working on charly/egress.go, the vendored schemas under
  charly/schema/vendor/, the ValidateEgress / registerVendoredEgressKind path,
  the offline `task cue:vendor` pipeline, or adding an egress schema for any
  written artifact (cloud-init, k8s manifests, traefik routes, runtime config,
  install ledger, systemd/quadlet units, ssh_config, libvirt XML).
---

# egress — validating the config files charly writes

## Overview — ingress vs egress

charly's CUE work has two halves:

- **Ingress** (`charly/cue_schema.go`): validates the INPUT config a user authors
  (`charly.yml` / box / candy / vm / k8s / deploy) against `schema/*.cue`. Owned by
  `/charly-build:validate`.
- **Egress** (`charly/egress.go`, this skill): validates the OUTPUT config charly
  WRITES onto a system — the seed ISO's cloud-init, Kustomize manifests, quadlet
  and systemd units, the ssh_config fragment, the install ledger, libvirt domain
  XML, … — BEFORE the bytes are written. Ingress proves the input; egress proves
  the output. A malformed artifact fails loudly in charly with a precise CUE
  error instead of silently on the target.

## Two-layer model (what CUE can and cannot do)

CUE operates on structured data (YAML/JSON) and strings — it has no parser for
Dockerfile / systemd-INI / ssh_config. So egress validation is layered:

1. **Structured YAML/JSON outputs** (cloud-init, k8s, traefik, runtime config,
   ledger) validate DIRECTLY — ingest the rendered bytes, unify with the schema.
2. **Non-data text outputs** (quadlet, systemd units, shell rc, ssh_config,
   Containerfile) validate via their **structured pre-image** (the Go model just
   before rendering) plus an optional `text`-encoding constraint check on the
   rendered string (no unresolved `${VAR}`, required sections present).
3. **XML** (libvirt) validates via CUE's experimental `xml+koala` decode.

## The validator API (`charly/egress.go`)

| Function | Purpose |
|----------|---------|
| `ValidateEgress(kind, label, data []byte) error` | Ingest serialized YAML/JSON bytes, unify with the egress kind's schema, `Validate(cue.Concrete(true))`. JSON is a YAML subset, so one ingest path covers both. |
| `ValidateEgressValue(kind, label, v any) error` | Validate an in-memory Go value (a manifest `map[string]any`, a record struct) — `cueSchemaCtx.Encode(v)` then unify+validate, no marshal roundtrip. Used where the writer holds the artifact as a Go value just before serialization (k8s manifests). |
| `registerVendoredEgressKind(kind, file, defPath)` | Compile a vendored schema file as its OWN `cue.Value` and register it (see "schema sources"). |
| `egressDef(kind) (cue.Value, bool)` | Resolve a kind's def — the vendored registry first, then charly's own shared-scope kinds via `cueKindDef`. |

A failed `ValidateEgress` returns the CUE error (`errors.Details`) and the caller
aborts the write — the gate is always-on (no opt-in flag), consistent with
charly's "CUE owns validation, hard-error" rule.

## Schema sources — the separate-compile constraint (load-bearing)

`charly/cue_schema.go`'s `sharedCueSchema` **string-concatenates the package-less
`schema/*.cue` files** into ONE `CompileString` ("one shared scope"). So:

- **charly's OWN egress schemas** are written package-less under `charly/schema/`
  (e.g. `schema/egress_cloud_init.cue` → `#CloudInitMeta`, `#NetworkConfigV2`),
  registered with the ordinary `registerCueKind`, and resolved by `egressDef`'s
  `cueKindDef` fallback.
- **VENDORED schemas** (an upstream JSON Schema run through `cue import
  jsonschema:`) carry a `package` clause + CUE-stdlib `import (...)`, so they
  **cannot** be concatenated into that blob. Each lives under
  `charly/schema/vendor/`, compiles as its OWN instance via `CompileBytes` (the
  CUE-stdlib imports resolve in the bare `cueSchemaCtx` with no module loader),
  and registers via `registerVendoredEgressKind`. The vendored tree is embedded by
  `//go:embed schema/vendor/*.cue` in `egress.go`; `schema/*.cue` is non-recursive
  so the vendor tree never pollutes `sharedCueSchema`.

## The offline vendoring pipeline (`task cue:vendor`)

Vendoring keeps charly a hermetic single binary — schemas are converted to plain
`.cue` at DEV time and embedded; nothing is fetched at runtime. Requires the `cue`
CLI (the `/charly-tools:cue` candy):

1. pin the upstream source under `charly/schema/vendor/sources/`;
2. `cue import -f -p schema -l '#<Def>:' -o charly/schema/vendor/<name>.cue jsonschema: <source>`;
3. (curated registry modules) `cue mod get cue.dev/x/...` then extract the needed defs;
4. the generated `.cue` is committed pristine, so a reproducibility test can match `cue import` output.

`charly/schema/vendor/README.md` records the exact regen commands + pinned versions.

## What's validated today

| Artifact | Writer | Kind | Schema |
|----------|--------|------|--------|
| cloud-init **user-data** | `RenderCloudInit` (`cloud_init_render.go`) | `cloud_config` | vendored Canonical cloud-config (`schema/vendor/cloud_config.cue`, `#CloudConfig`) |
| cloud-init **meta-data** | `RenderCloudInit` | `cloud_init_meta` | `schema/egress_cloud_init.cue` `#CloudInitMeta` |
| cloud-init **network-config** | `RenderCloudInit` | `cloud_init_net` | `schema/egress_cloud_init.cue` `#NetworkConfigV2` |
| **k8s manifests** (Deployment/StatefulSet/DaemonSet/Job/CronJob/Pod/Service/PVC/Ingress) | `GenerateK8sKustomize` → `writeK8sYAML` (`k8s_generate.go`) | `k8s_object` | `schema/egress_k8s.cue` `#K8sObject` envelope (validates structure — the egress failure mode for machine-generated manifests; deep per-field types are an ingress concern) |
| **k8s Kustomization** (base + overlay) | `GenerateK8sKustomize` → `writeK8sYAML` | `kustomization` | `schema/egress_k8s.cue` `#Kustomization` |
| **install-ledger deploy record** | `WriteDeployRecord` / `WriteDeployRecordVia` (`install_ledger.go`) | `deploy_record` | `schema/egress_ledger.cue` `#DeployRecord` (requires `deploy_id`/`target`/`deployed_at`; `image` optional — candy-only deploys leave it empty) |
| **install-ledger candy record** | `WriteCandyRecord` / `AddCandyDeploymentVia` | `candy_record` | `schema/egress_ledger.cue` `#CandyRecord` (requires `candy`/`deployed_at`; steps/reverse_ops open) |
| **traefik dynamic config** (`.build/<box>/traefik-routes.yml`) | `generateTraefikRoutes` (`generate.go`) | `traefik_routes` | `schema/egress_traefik.cue` `#TraefikRoutes` (hand-built YAML — non-empty Host rule / service / backend url; null routers/services when no route candies) |

## Deliberately NOT egress-validated

The gate earns its keep on output that is HAND-ASSEMBLED with real invariants
(traefik routes, ledger records) or that incorporates external/variable data
(cloud-init Extra, k8s capabilities from labels). It is **intentionally not added**
where the writer is a straight `yaml.Marshal(typedStruct)` of charly's own struct
with no transformation — the output cannot malform by construction, so a schema
would be validation theater:

- **runtime config** `~/.config/charly/config.yml` (`SaveRuntimeConfig` — pure `yaml.Marshal(*RuntimeConfig)`).
- **deploy-state** `~/.config/charly/charly.yml` (`SaveDeployConfig`) — additionally, project config is already CUE-validated on LOAD (ingress).

## Caveats

- **`matchN` disables closedness.** A schema imported from a Draft-04 JSON Schema
  (cloud-config) uses `matchN` (anyOf), which means CUE does NOT reject unknown
  keys — it catches TYPE/structure violations, not typos. Acceptable for egress
  (validating output charly itself generated). charly's hand-authored egress
  schemas stay `close({...})`.
- **CUE reorders keys.** Any generation path (`cueyaml.Encode`) emits keys sorted —
  output is SEMANTICALLY equal to a hand-marshal, never byte-identical. Round-trip
  tests must parse-and-compare.

## Adding a new egress schema (recipe)

1. Decide the schema source: hand-author package-less under `schema/` (register
   with `registerCueKind`) OR vendor an upstream schema under `schema/vendor/`
   (register with `registerVendoredEgressKind`, add to `task cue:vendor`).
2. Add a one-line registration file (mirror `cue_egress_cloud_config.go` /
   `cue_kind_cloud_init_egress.go`).
3. Call `ValidateEgress(kind, label, bytes)` at the writer's seam, BEFORE the
   `os.WriteFile` / `PutFile`, returning its error.
4. Add corpus + teeth tests (a good artifact passes, a malformed one is rejected) —
   mirror `egress_test.go`.

## Cross-References

- `/charly-tools:cue` — the `cue` CLI candy that drives the vendoring pipeline.
- `/charly-build:validate` — the ingress side (`charly box validate`).
- `/charly-internals:cloud-init-renderer` — the first egress consumer (`RenderCloudInit`).
- `/charly-internals:install-plan` + `/charly-internals:local-infra` — the DeployTargets/writers whose seams the egress gate plugs into.
- `/charly-internals:go` — the Go source map (`egress.go`, `cue_schema.go`).

## When to Use This Skill

Invoke before working on `charly/egress.go`, the vendored schemas under
`charly/schema/vendor/`, the `ValidateEgress` / `registerVendoredEgressKind` path,
the `task cue:vendor` pipeline, or adding/validating any config file charly writes
to a system.
