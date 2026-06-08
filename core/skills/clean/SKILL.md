---
name: clean
description: |
  Prune reusable build artifacts to defaults: retention (images, eval runs) and
  sweep one-time makepkg leftovers.
  MUST be invoked before any work involving: charly clean, build-artifact retention,
  keep_images / keep_eval_runs, image-tag pruning, or .eval run cleanup.
---

# charly clean -- Build-artifact retention + cleanup

## Overview

`charly clean` reclaims disk by applying the project's configured retention to
**reusable** build artifacts and removing **one-time** transient leftovers. It is
the on-demand counterpart to the auto-pruning that runs after `charly box build`
and `charly eval run`.

Two artifact classes, two policies (operator principle):

- **One-time / transient → always cleaned immediately.** makepkg leftovers under
  `pkg/arch` (`src/`, `pkg/`, `*.pkg.tar.zst`, `*.log`). `task build:ov` removes
  these right after install; `charly clean` sweeps any backlog.
- **Reusable → keep-last-N, configurable in `defaults:`.** Container image tags
  and `.eval` run output. Retention is set in `charly.yml` `defaults:` and
  applied automatically at creation; `charly clean` applies it on demand.

## Config (charly.yml `defaults:`)

```yaml
defaults:
  keep_images: 3      # newest CalVer tags to keep per image after `charly box build`
  keep_eval_runs: 3   # newest run dirs to keep per bed/score after `charly eval run`
```

`0` (or absent → built-in fallback `0`) **disables** that retention. The repo
opts in; third-party configs get no surprise pruning until they set a value.

## Commands

```bash
charly clean              # apply retention now: prune images + eval runs + makepkg leftovers
charly clean --dry-run    # print everything that WOULD be removed; touch nothing
charly clean --images     # only image-tag retention
charly clean --eval       # only eval-run retention
charly clean --keep N     # override the retention count for this run (0 = use defaults:)
```

With neither `--images` nor `--eval`, all three categories run (images + eval +
makepkg). `--keep N` overrides both counts for the invocation.

## What gets pruned (and what never does)

**Image-tag retention** (`keep_images`): images are grouped by the
`ai.opencharly.image` label and ordered by the `ai.opencharly.version`
CalVer label; all but the newest N per group are `podman rmi`'d. **Safety**: any
image referenced by a container (`podman ps -a`, including stopped/quadlet
deploys) is skipped, and `rmi` runs WITHOUT `-f` so the engine refuses any
still-referenced image as a backstop. Non-charly images (no `ai.opencharly.image`
label) and images with an unparseable version are never touched.

**Eval-run retention** (`keep_eval_runs`): each `.eval/<bed|score>/` dir is
trimmed to the newest N run artifacts — CalVer-named run dirs (bed runs),
`runs/<id>/` dirs (score iterations), and `result-<calver>.yml` files.
**`NOTES.md` is ALWAYS preserved** (it's the durable Syncthing-replicated harness
memory), as is any other non-run file.

**makepkg sweep**: removes `pkg/arch/{src,pkg,*.pkg.tar.zst,*.log}` (the package
is already installed via pacman — pure transient waste).

`--dry-run` is best-effort: it lists prune candidates but cannot see "external"
build containers (buildah intermediates `podman ps -a` doesn't list), so an
image held by one is listed yet safely skipped at removal time (the `rmi`
backstop refuses it). The real run silently retains such in-use images.

## Auto-prune at creation

The same retention runs automatically (no flag needed):

- After `charly box build` (push runs excluded) → `keep_images`.
- After `charly eval run` (any path: bed / `--all-beds` / score) → `keep_eval_runs`,
  after the new run's output is written so the newest run is kept.

`charly clean` exists for on-demand sweeps and to clear a pre-existing backlog.

## Out of scope

VM disk images (`output/`, `image/*/output/`) are single products per type
(overwritten on rebuild, not accumulated) — remove them on demand with
`charly vm destroy --disk`. The VM raw intermediate is already auto-cleaned during
the qcow2 build.

## Implementation

`ov/clean.go` — `pruneImagesByRetention`, `pruneEvalRuns`,
`cleanMakepkgArtifacts`, `CleanCmd`. Hooks in `BuildCmd.Run` (`ov/build.go`) and
`EvalRunCmd.Run` (`ov/eval_runner_cmd.go`). Retention keys live on `BoxConfig`
(`ov/config.go`), merged via `mergeImageConfig` (`ov/unified.go`), validated in
`validateBuildTunables` (`ov/validate.go`).

## Cross-References

- `/charly-build:build` — `charly box build` + the `keep_images` auto-prune.
- `/charly-eval:eval` — `charly eval run` + the `keep_eval_runs` auto-prune.
- `/charly-vm:vm` — `charly vm destroy --disk` for VM disk removal.
- `/charly-image:image` — the `defaults:` block where retention keys live.
