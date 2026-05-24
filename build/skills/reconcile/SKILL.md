---
name: reconcile
description: |
  Use when @github layer/namespace pins drift across repos and the resolver emits
  "referenced at multiple versions" warnings — `ov image reconcile` aligns every
  pin of a repo to one version (clearing the warnings). Invoked as
  `ov image reconcile`.
---

# ov image reconcile — align cross-repo `@github` version pins

Invoked as `ov image reconcile`. See `/ov-image:image` for the family overview.

## Overview

The layer resolver compares each layer's PER-ENTITY `version:` (read after fetch),
not the repo git tag — so a repo re-tag of an UNCHANGED layer does NOT warn. When
a layer DOES resolve to two different per-entity versions (a family pinned to a
genuinely newer layer than the shared infra it composes), it **warns once and uses
the newest** (see `/ov-internals:go` "Remote-layer resolver", `/ov-build:validate`).
The git `:vTAG` is only the FETCH coordinate. `ov image reconcile` aligns the
on-disk git-tag pins — for each distinct repo referenced by the project's
versioned YAML, it rewrites EVERY pin of that repo to ONE target tag, so every
reference fetches one commit per repo and the next `ov image generate` emits
**zero** version warnings. Edits are comment-preserving (yaml.v3 node API) and idempotent.

The **zero-warnings R10 gate** (CLAUDE.md R1) makes this load-bearing: a change
that introduces a version mismatch is not landable until `ov image reconcile`
clears the warning.

## Quick Reference

| Action | Command | Description |
|---|---|---|
| Preview rewrites | `ov image reconcile --dry-run` | Print every pin it would change; touch nothing |
| Align to newest referenced | `ov image reconcile` | Rewrite each repo's pins to the newest version ALREADY referenced (offline) |
| Align to newest remote tag | `ov image reconcile --remote` | Query `git ls-remote --tags` per repo and bump to the newest tag |

```bash
ov image reconcile --dry-run      # see the plan
ov image reconcile                # align to newest referenced (no network)
ov -C image/selkies image reconcile   # reconcile a submodule's pins
```

## What it does

1. **Scan** every `@github.com/owner/repo[/path]:vTAG` ref in the project's
   versioned YAML files (`overthink.yml` + flat-imported `image.yml` / `base.yml` /
   `eval.yml` / `local.yml` / `build.yml` / `pod.yml` / `k8s.yml` / `vm.yml` /
   `deploy.yml`). Refs appear in `import:` namespaces, image `base:` / `builder:` /
   `layer:`, and `kind:local` `layer:` lists.
2. **Target** per repo: the newest version ALREADY referenced (default —
   `compareSemver`, orders CalVer correctly, no network) or, with `--remote`, the
   newest tag on the remote (`GitLatestTag`).
3. **Rewrite** every pin of that repo whose version differs from the target,
   preserving comments and key order. Unpinned refs and single-version repos are
   left untouched. Idempotent: a second run rewrites nothing.

## Scope — one project per invocation

`ov image reconcile` operates on the CURRENT project (cwd; honors the top-level
`-C` / `--dir` / `OV_PROJECT_DIR`). For a multi-repo tree (the main repo + its
`image/<distro>` submodules), run it per repo, or per submodule via `-C
image/<name>`. This pairs with the cross-repo landing order in
`/ov-internals:git-workflow` B6: land + tag the producer FIRST, then
`ov image reconcile` repoints the consumer to the producer's fresh tag before the
consumer's authoritative R10.

## Implementation

`ov/reconcile.go` — `ImageReconcileCmd` (wired under `ov image` in `ov/image.go`).
Reuses `ParseRemoteRef` / `IsRemoteLayerRef` / `StripVersion` (`ov/refs.go`),
`compareSemver` / `GitLatestTag` / `RepoGitURL` (`ov/refs_git.go`), and the
comment-preserving load/`yaml.Marshal` pattern from `ov/yaml_setter.go`. Covered
by `ov/reconcile_test.go` (newest-referenced alignment, comment preservation,
idempotency, single-version-untouched, no-pins no-op).

## Cross-References

- `/ov-internals:go` "Remote-layer resolver" — the warn-and-newest-wins resolver this aligns to.
- `/ov-build:validate` — surfaces the multi-version warning reconcile clears.
- `/ov-internals:git-workflow` — cross-repo (B6) producer→consumer landing that calls reconcile.
- `/ov-build:migrate` — per-push CalVer tags that reconcile pins point at.
- `/ov-image:image` — `import:` / namespace authoring + the family overview.

## When to Use This Skill

Invoke when the resolver warns that a layer is referenced at multiple versions,
when aligning a consumer's pins to a freshly-tagged producer, or whenever you need
the project's `@github` pins consistent for a zero-warnings R10.
