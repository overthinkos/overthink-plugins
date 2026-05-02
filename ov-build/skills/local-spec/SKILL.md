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

There is **no** `status:` or `info:` field — those were removed in the local-cutover. Status lives in `description.tag` (one of `working`/`testing`/`broken`); the human-facing description lives in `description.feature` + `description.narrative`.

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
- `/ov-build:migrate` — `ov migrate target-local` migrates legacy `kind: host`/`host.yml` projects.

## When to Use This Skill

**MUST be invoked** when the task involves authoring or editing `kind: local` templates, `local.yml` files, the inline `local:` map, or the merge between a template and a deployment. Invoke this skill BEFORE reading Go source or launching Explore agents.
