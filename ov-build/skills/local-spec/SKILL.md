---
name: local-spec
description: |
  MUST be invoked before any work involving: authoring `kind: local` templates (the post-cutover replacement for `kind: host`), `local.yml` files, the inline `local:` map in `overthink.yml`, or the merge semantics between a `kind: local` template and a `target: local` deployment.
---

# kind: local — Authoring Reference

## Overview

`kind: local` declares a reusable layer-stack template that gets applied to a Linux filesystem (target:local deployments). Unlike `kind: pod` / `kind: vm` / `kind: k8s` which wrap an image, a `kind: local` is defined entirely by its `layers` + `install_opts` + `env` — there's no OCI artifact backing it. The convention file is `local.yml`; templates may also be authored inline in the `local:` map of `overthink.yml`.

Replaces the legacy `kind: host`. Migration: `ov migrate target-local`.

## Schema

```yaml
kind: local
name: dev-workstation
spec:
  layers:        # required (use [] for a placeholder; see below)
    - ripgrep
    - direnv
    - uv
  install_opts:  # optional — defaults merged 3-tier (CLI > deployment > template)
    with_services: false
    allow_repo_changes: true
  env:           # optional
    - EDITOR=vim
  description:   # optional — Gherkin-shaped self-description
    feature: Standard developer workstation profile
    tag: [working]
  eval:          # optional — deploy-scope checks
    - {id: rg-version, scope: deploy, command: "rg --version"}
```

## Inline form in overthink.yml

```yaml
version: 4
local:
  dev-workstation:
    layers: [ripgrep, direnv]
    install_opts: {with_services: false, allow_repo_changes: true}
    env: [EDITOR=vim]
    description: {feature: Dev workstation, tag: [working]}

  ci-runner:
    layers: [ripgrep, pixi, cargo-toolchain]
    install_opts: {with_services: true, allow_root_tasks: true}
    description: {feature: CI runner profile, tag: [working]}

  cachyos-dx:
    # Empty placeholder — `layers: []` is a load-time WARNING (allowed
    # for staged template name reservation); a missing `layers:` field
    # is a hard error.
    layers: []
    install_opts: {}
    description: {feature: CachyOS DX (placeholder), tag: [testing]}
```

## Field reference

| Field | Required | Description |
|---|---|---|
| `layers` | Yes | Ordered layer stack. `[]` permitted as a placeholder (warning, not error). |
| `install_opts` | No | Default install gates. Deployment overrides merge on top. |
| `env` | No | Shell-profile env vars (`KEY=VALUE`). Deployment env wins on key collision. |
| `description` | No | Gherkin-shaped (Feature/Narrative/Tag/Scenario). Status word lives in `tag`: `working`/`testing`/`broken`. |
| `eval` | No | Deploy-scope checks; merged with deployment.eval. |
| `deploy_eval` | No | Same as `eval` but reserved for image-style deploy-tests propagation. |
| `images` | No | Container images that must be present in local podman storage before tests can run. See "Images Surface" below. 2026-05 cutover. |

There is **no** `status:` or `info:` field — those were removed in the local-cutover. Status lives in `description.tag` (one of `working`/`testing`/`broken`); the human-facing description lives in `description.feature` + `description.narrative`.

## Images Surface (`images:`)

A `kind: local` template can declare the container images it needs
present in local podman storage before tests can run. Without this,
`ov eval run` against a freshly-deployed host fails with "image not
local" because the deploy installs **layers** (host packages + configs)
not **images** (container artifacts).

```yaml
local:
  cachyos-dx:
    layers: [...]
    images:
      - eval-target               # short name → cfg.Images[eval-target] → ghcr.io/.../eval-target:<tag>
      - openclaw-sway-browser     # short name
      - fedora-coder              # short name
    install_opts: {...}
```

Three input forms (mirror `ov image pull`):

| Form | Example | Resolution |
|---|---|---|
| Short name | `eval-target` | Resolved via `cfg.Images[eval-target]` to `<registry>/<name>:<tag>` |
| Fully-qualified registry ref | `ghcr.io/overthinkos/eval-target:2026.124.1253` | Pass-through |
| Remote project ref | `@github.com/overthinkos/overthink/eval-target:latest` | Resolves the repo, reads its `image.yml`, returns the declared registry ref |

**Pull-first, build-fallback** — at deploy time, for each entry:

1. `LocalImageExists(engine, ref)` short-circuit — already present, no-op.
2. Try `ov image pull <ref>` (preferred — fast, no compile). Routed
   through the `DeployExecutor` so SSH-routed deploys (`host:
   user@machine`) pull on the **remote** machine.
3. If pull fails AND the image is a short-name that resolves to
   `cfg.Images[<name>]`, fall back to `ov image build <name>`. Build
   fallback is local-only — SSH executors return an actionable error
   asking the operator to pre-build the image.

**Validation at `ov image validate`** (`validateLocalImages` in
`ov/validate.go`):
- Short names: must exist in `cfg.Images`. Suggests similar names on
  miss via `findSimilarName`.
- Fully-qualified refs: syntax-valid registry ref check.
- `@github.com/...`: syntax-valid remote ref check.
- Duplicates within one template: hard error.

**Idempotent.** Running `ov deploy add cachyos-dx` twice on the same
host — when all images are already present — emits "image=X present"
for each declared entry and is a no-op for podman.

**Refcount-aware cleanup at `ov deploy del`.** Each `EnsureImageStep`
records a `ReverseOpRemoveImage` op tagged with the deploy ID. Image
removal happens ONLY when all of:

1. The last deploy referencing the image is being torn down (refcount
   reaches zero in the install ledger), AND
2. `ov deploy del --reclaim-images` is passed.

Default behaviour leaves images in podman storage across deploys
because pulls are expensive. The `--reclaim-images` flag is opt-in.

**Failure mode** — if pull fails AND fallback build fails (or isn't
applicable for the executor), the deploy aborts BEFORE any layer
plan walks. Operators see the failure early; partial deploys are
avoided.

**IR step kind** — `StepKindEnsureImage` / `EnsureImageStep` in
`ov/install_plan.go`. Compiled by `compileImagesSteps` in
`install_build.go`. Emitted by `LocalDeployTarget.execEnsureImage` in
`ov/deploy_target_local.go` as a pre-pass before the layer plan walk.

## Merge semantics — `kind: local` template + `kind: deployment`

When a deployment carries `local: <template-name>`:

| Field | Template provides | Deployment overrides | CLI overrides | Effective value |
|---|---|---|---|---|
| `layers` | base list | — | — | `template.Layers` |
| `add_layers` | — | extra list | `--add-layer` | `deployment.AddLayers ++ CLI.AddLayer` |
| effective layer order | — | — | — | `template.Layers ++ deployment.AddLayers ++ CLI.AddLayer` |
| `install_opts.*` (bool) | default | wins over template | wins over both | CLI > deployment > template |
| `install_opts.builder_image` | default `""` | wins | wins over both | CLI > deployment > template |
| `env` | base list | extends + overrides on key | — | template.Env merged with deployment.Env (deployment-wins on collision) |
| `eval` | base checks | extends list | — | `template.Eval ++ deployment.Eval` |
| `deploy_eval` | base checks | extends list | — | `template.DeployEval ++ deployment.DeployEval` |

The `InstallOptsConfig.ApplyTo` method is fill-empty — calling it on the deployment's opts first, then the template's, gives the priority chain automatically.

## Empty `layers: []` is a placeholder

A template with `layers: []` is permitted as a stub for staged name reservation:

```yaml
local:
  cachyos-dx:
    layers: []
    install_opts: {}
    description: {feature: CachyOS DX (placeholder), tag: [testing]}
```

`ov image validate` emits a WARNING but does not error. A missing `layers:` field entirely IS an error — the field's presence is the signal that the author intended a template.

## Cross-References

- `/ov-advanced:local-deploy` — the `target: local` deployment surface that consumes this template.
- `/ov-dev:local-infra` — Go file map (`local_spec.go`, `LocalSpec` struct, `findLocalSpec` lookup).
- `/ov-build:layer` — layer authoring (the building blocks composed by templates).
- `/ov-build:migrate` — `ov migrate target-local` migrates legacy `kind: host`/`host.yml` projects; `ov migrate qc-rename` renames the operator-specific `qc` deployment key to `cachyos-dx` (demonstrating that a kind:local template and a kind:deployment entry can share a name — cross-kind reuse, 2026-05-05).

## Cross-kind name reuse (2026-05-05)

A `kind: local` template's name lives in the `local:` namespace, independent of layer / image / pod / vm / k8s / deployment. The canonical example is `cachyos-dx` — `local.cachyos-dx` is the template; `deployment.cachyos-dx` is the deployment entry that applies it; both share the name without conflict. Verbs disambiguate: `ov rebuild cachyos-dx` resolves to the deployment entry; the template is referenced internally via the deployment's `local: cachyos-dx` field. See CLAUDE.md "Cross-kind name reuse is permitted and encouraged".

## When to Use This Skill

**MUST be invoked** when the task involves authoring or editing `kind: local` templates, `local.yml` files, the inline `local:` map, or the merge between a template and a deployment. Invoke this skill BEFORE reading Go source or launching Explore agents.
