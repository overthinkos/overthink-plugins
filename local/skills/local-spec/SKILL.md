---
name: local-spec
description: |
  MUST be invoked before any work involving: authoring `kind: local` templates, `local.yml` files, the inline `local:` map in `charly.yml`, or the merge semantics between a `kind: local` template and a `target: local` deployment.
---

# kind: local — Authoring Reference

## Overview

`kind: local` declares a reusable candy-stack template that gets applied to a Linux filesystem (target:local deployments). Unlike `kind: pod` / `kind: vm` / `kind: k8s` which wrap an image, a `kind: local` is defined entirely by its `layers` + `install_opts` + `env` — there's no OCI artifact backing it. The convention file is `local.yml`; templates may also be authored inline in the `local:` map of `charly.yml`.

Legacy `kind: host` projects migrate via `charly migrate`.

## Schema

```yaml
kind: local
name: dev-workstation
spec:
  candy:        # required (use [] for a placeholder; see below)
    - ripgrep
    - direnv
    - uv
  install_opts:  # optional — defaults merged 3-tier (CLI > deployment > template)
    with_services: false
    allow_repo_changes: true
  env:           # optional
    - EDITOR=vim
  description: Standard developer workstation profile   # optional — plain string
  plan:          # optional — flat ordered list of steps; each is one intent keyword + inline Op
    - check: ripgrep is installed after deploy
      command: rg --version
      context: [deploy]
```

## Inline form in charly.yml

```yaml
version: 2026.144.1443
local:
  dev-workstation:
    candy: [ripgrep, direnv]
    install_opts: {with_services: false, allow_repo_changes: true}
    env: [EDITOR=vim]
    description: Dev workstation

  ci-runner:
    candy: [ripgrep, pixi, cargo-toolchain]
    install_opts: {with_services: true, allow_root_tasks: true}
    description: CI runner profile

  charly-cachyos:
    # Empty placeholder — `candy: []` is a load-time WARNING (allowed
    # for staged template name reservation); a missing `candy:` field
    # is a hard error.
    candy: []
    install_opts: {}
    description: CachyOS DX (placeholder)
```

## Field reference

| Field | Required | Description |
|---|---|---|
| `layers` | Yes | Ordered candy stack. `[]` permitted as a placeholder (warning, not error). |
| `install_opts` | No | Default install gates. Deployment overrides merge on top. |
| `env` | No | Shell-profile env vars (`KEY=VALUE`). Deployment env wins on key collision. |
| `description` | No | Plain string — the profile's purpose; first line = the summary. |
| `plan` | No | Flat ordered list of steps (sibling of `description`). Each step is one intent keyword — `run:` (state-change) / `check:` (deterministic probe) / `agent-run:` / `agent-check:` / `include:` — carrying prose plus, for `run:`/`check:`, an inline Op (verb + matchers + `context:`). Deploy-scope steps carry `context: [deploy]`. Merged with deployment.plan. |

There is **no** `status:` or `info:` field; `description` is the human-facing summary string.

## What the deploy does NOT do

A `kind: local` deploy emits **zero** container-image fetch / build
steps. The deploy applies host packages + configs only. There is no
`image:` field; declaring one in legacy YAML hard-errors at
`charly box validate` time with a pointer to `charly migrate`.

Test-bed image preflight is the **check entry point's** job, not the
deploy's. When `charly check run --on-host <name>` resolves to a host
target, the runner walks the bed's `plan:`, collects every distinct
step `pod:` value plus the bed's target image, and ensures each
image is present in local podman storage (LocalImageExists →
`charly box pull` → fall back to `charly box build` for short names that
resolve via `cfg.Images`). Operators who never run `charly check run`
never pay the image-fetch cost. See `/charly-check:check` "Image
preflight" and `charly/check_image_preflight.go`.

This invariant — "deploy fetches NOTHING speculative" — is codified
as a CLAUDE.md Key Rule and enforced at the type level: the
`LocalSpec` Go struct has no `Images` field, so the surface is
unreachable from any new code. Migration: `charly migrate`
(idempotent; rewrites legacy `image:` blocks under `local.<name>`
to a dated comment fence).

## Merge semantics — `kind: local` template + `kind: deploy`

When a deployment carries `local: <template-name>`:

| Field | Template provides | Deployment overrides | CLI overrides | Effective value |
|---|---|---|---|---|
| `layers` | base list | — | — | `template.Layers` |
| `add_layers` | — | extra list | `--add-candy` | `deployment.AddCandies ++ CLI.AddCandy` |
| effective layer order | — | — | — | `template.Layers ++ deployment.AddCandies ++ CLI.AddCandy` |
| `install_opts.*` (bool) | default | wins over template | wins over both | CLI > deployment > template |
| `install_opts.builder_image` | default `""` | wins | wins over both | CLI > deployment > template |
| `env` | base list | extends + overrides on key | — | template.Env merged with deployment.Env (deployment-wins on collision) |
| `plan` | base plan | extends list | — | `template.Plan ++ deployment.Plan` |

The `InstallOptsConfig.ApplyTo` method is fill-empty — calling it on the deployment's opts first, then the template's, gives the priority chain automatically.

## Empty `candy: []` is a placeholder

A template with `candy: []` is permitted as a stub for staged name reservation:

```yaml
local:
  charly-cachyos:
    candy: []
    install_opts: {}
    description: CachyOS DX (placeholder)
```

`charly box validate` emits a WARNING but does not error. A missing `candy:` field entirely IS an error — the field's presence is the signal that the author intended a template.

## Cross-References

- `/charly-local:local-deploy` — the `target: local` deployment surface that consumes this template.
- `/charly-internals:local-infra` — Go file map (`local_spec.go`, `LocalSpec` struct, `findLocalSpec` lookup).
- `/charly-image:layer` — candy authoring (the building blocks composed by templates).
- `/charly-build:migrate` — `charly migrate` migrates legacy `kind: host`/`host.yml` projects and splits `charly.yml`'s inline `image:` / `vm:` / `pod:` / `k8s:` / `local:` / `deploy:` maps into sibling per-kind files. The `charly-cachyos` deploy key + `local.charly-cachyos` template share a name — a concrete demonstration of cross-kind name reuse (a `kind: local` template and a `kind: deploy` entry can share a name).

## Cross-kind name reuse

A `kind: local` template's name lives in the `local:` namespace, independent of layer / image / pod / vm / k8s / deploy. The canonical example is `charly-cachyos` — `local.charly-cachyos` is the template; `deploy.charly-cachyos` is the deployment entry that applies it; both share the name without conflict. Verbs disambiguate: `charly update charly-cachyos` resolves to the deploy entry; the template is referenced internally via the deploy's `local: charly-cachyos` field. See CLAUDE.md "Cross-kind name reuse is permitted and encouraged".

## When to Use This Skill

**MUST be invoked** when the task involves authoring or editing `kind: local` templates, `local.yml` files, the inline `local:` map, or the merge between a template and a deployment. Invoke this skill BEFORE reading Go source or launching Explore agents.
