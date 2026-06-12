---
name: git-workflow
description: |
  Use when committing, branching, pushing, merging, tagging, creating PRs, or
  approving/merging PRs with gh — the feat/-branch, R10-gated, never-force-push
  landing workflow across the main repo + the plugins submodule + box/<distro>
  submodules. Covers sync-to-upstream, branch/worktree pruning, the fork+PR path
  for contributors without write access, and cross-repo @github landing order.
---

# git-workflow — branch-per-change, R10-gated auto-landing

Every change to an charly-project repo follows ONE landing discipline. The **R10
pass is the sole gate**: nothing is committed, pushed, merged, or tagged on
unverified state, and once R10 passes the landing is automatic — no per-change
manual "push" step. This skill is the mechanics; CLAUDE.md "Post-Execution
Policies" carries the mandate, `/charly-internals:cutover-policy` the one-phase rule,
`/charly-build:migrate` the schema-version/tag coupling.

## Non-negotiable invariants

- **NEVER force-push.** No `git push --force`, no `--force-with-lease`, on ANY
  branch (`feat/` included) in ANY repo, ever. The whole flow is designed so a
  force push is never needed: `main` only fast-forwards; tags are add-only;
  `feat/` is pushed once at landing.
- **R10-gated.** Commit/push/merge/tag happen only after R10 PASS. A rule
  violation or R10 FAIL ⇒ none of them happen (fix in the same tree, re-run R10).
- **Zero warnings.** R10 is NOT successful while ANY warning remains — resolver
  newest-wins warnings, build, `charly box validate`, `charly eval`, or deploy
  warnings. Every warning is fixed before R10 passes (a version-mismatch warning
  is cleared with `charly box reconcile`; any other warning triggers
  `/charly-internals:root-cause-analyzer` then a real fix). "Warning" is never an
  acceptable end state — it is an R10 failure (strengthens R1).
- **Atomic.** One commit per repo per cutover (the cutover-policy "one phase").
- **Tree-safety before destructive actions (R6).** Always check `git status` +
  `git stash list` before any destructive working-tree action — `git stash`
  discards in-progress work; `rm` on a tracked file is destructive. When the
  sandbox blocks an action, read the reason and find a non-destructive
  alternative — never work around it with a cleverer command.
- **Eval-coverage.** R10 does not pass unless the change ships the test coverage
  that PROVES its functionality (`eval:` checks for new/changed layers & images,
  Go tests for `charly` code) AND the live run exercised it. A change whose new
  functionality has no test that would FAIL without it is not landable.
- **Tags only on `charly.yml` repos.** `plugins` and `pkg/arch` are tag-exempt.

## B1 — the branch-per-change loop (write access)

```bash
# sync-before-start (see B4): branch off up-to-date main
git fetch origin --prune --tags
git switch main && git merge --ff-only origin/main
git switch -c feat/<slug>            # slug = kebab summary of the change

# ... implement the whole cutover; run beds freely throughout to VERIFY
#     (verify before you change — Risk Driven Development: prove high-risk
#     assumptions on a bed first); the COMMIT is gated on the full final-code
#     live test (pasted), which runs at the end ...

# on R10 PASS, automatically and in order:
git add <only the cutover's files>   # never the in-flight state of unrelated work
git commit -m "<conventional commit>  ...  Assisted-by: Claude (<tier>)"
git push origin feat/<slug>
git switch main && git merge --ff-only origin/main   # re-sync; if main advanced, see below
git merge --ff-only feat/<slug>
git tag -a "$(date -u +v%Y.%j.%H%M)" -m "<subject>" HEAD
git push origin main --follow-tags
git branch -d feat/<slug> && git push origin --delete feat/<slug>
```

If `main` advanced between branch-start and landing, the `--ff-only` merge refuses
(that's the safety property — never a force). Rebase `feat/` onto the new `main`,
**re-run R10**, then ff-merge. `feat/` is pushed once at landing, so a
rebase-then-push collision can't normally arise; in the rare case `feat/` was
already pushed and then rebased, update the remote with
`git push --delete origin feat/<slug>` followed by a fresh push — NEVER `--force`.

## B4 — sync to upstream + prune (per repo: main, plugins, box/*)

- **Sync-before-start / before-landing.** `git fetch origin --prune --tags`; ff
  local `main` to `origin/main`. Never force-reset a diverged local `main` — if it
  cannot fast-forward, STOP + run `/charly-internals:root-cause-analyzer`.
- **Switch-to-upstream check.** Before committing, confirm `origin` is the
  canonical upstream and the branch about to merge targets the upstream `main`
  (not a stale fork/branch). On mismatch, STOP and surface it.
- **Prune merged branches.** `feat/` is deleted at landing (B1). Sweep leftovers:
  `git branch --merged main` → delete local; `git fetch --prune` drops
  remote-tracking refs deleted upstream; `git branch -r --merged origin/main` →
  `git push origin --delete` the merged remote `feat/*`. **Only ever delete
  branches confirmed `--merged`**; never `-D` (force-delete) an unmerged/abandoned
  branch without operator confirmation — it may hold unlanded work.
- **Worktree hygiene.** `git worktree list` to inventory; `git worktree prune` to
  clear stale admin entries. Remove an agent `isolation: worktree` after its
  change lands. Before reusing a long-lived worktree, ff its base to `origin/main`.

## B2 — multi-repo / multi-worktree coordination

One logical change spanning several repos uses the **same `feat/<slug>` in each**
(main, `plugins`, `box/<distro>`), so the branches correlate. R10 runs against the
**assembled superproject** (submodule pointers at the `feat/` commits) — the whole
change is verified before anything merges. Then land in **dependency order**:

1. each `box/<distro>` submodule — commit → `--ff-only` merge → tag (it has
   `charly.yml`) → push;
2. `plugins` — commit → `--ff-only` merge → push (**no tag**, no `charly.yml`);
3. the superproject — stage the now-merged submodule pointers → atomic commit →
   `--ff-only` merge → tag `main` → push.

**Submodule-pointer-bump safety (step 3) — bump AFTER the switch, then stage AND
verify.** A `git switch` / `git checkout` re-materializes each submodule at the
gitlink the *target branch* records, silently discarding an **unstaged**
working-tree pointer bump (it happens even with `submodule.recurse` unset — an
unstaged gitlink is not carried across the switch). So bumping the pointer
*before* `git switch -c feat/<slug>` — or merely `git -C <sub> checkout <new>`
without `git add` — drops it from the commit, and a `git add <sub>; git commit`
afterward stages nothing because the working tree was reset to the old pointer.
Always, in order: (a) create/switch to the landing branch FIRST; (b) THEN
`git -C <sub> checkout <new-commit>` + `git add <sub>`; (c) VERIFY it is staged —
`git diff --cached --submodule=short <sub>` must print `<old>...<new>`; (d) after
committing, confirm the commit records it — `git show --stat` lists `<sub>` and
`git ls-tree HEAD <sub>` shows `<new>`. A pointer-bump commit whose `--stat` omits
the submodule is the silent-drop failure. If it was already pushed, land a NEW
pointer-bump commit (NEVER amend/force-push). See `CHANGELOG.md` 2026-06-08 for
the incident this rule prevents.

This mirrors the submodules-first push order. A change developed in a git worktree
keeps its `feat/` branch in the worktree; the ff-merge targets the canonical
repo's `main`; the worktree is removed after. Concurrent worktrees on one repo
each use a distinct `feat/` slug.

## B3 — agent teams on ONE shared tree (no worktree)

When an agent team parallelizes work, **the eval bed is the unit of isolation,
not a worktree**. Each teammate owns a disjoint `kind: eval` bed's SOURCE files;
distinct beds get distinct container/VM/image names; the lead assigns each
disjoint host ports too (the loader does NOT check ports — an overlap fails the
second bed at deploy), and a bed pins an image → layers → files, so bed-ownership
already isolates the source files each teammate edits. **Teammates edit; a
PERSISTENT owner runs every full `charly eval run <bed>`** as a `run_in_background`
task — the lead's persistent session, a background agent, or (interactive tmux) a
split-pane teammate; an in-process teammate CANNOT (its bg dies on yield).
Teammates therefore share ONE working tree on ONE `feat/<slug>` branch:

- Teammates edit their bed-scoped files in the shared tree + run short foreground
  checks (`charly eval box`) — never the full `charly eval run`, and **never commit or
  push**. The lead runs the full beds and owns the single atomic commit, gated on
  the consolidated full final-code bed run (B1).
- Reserve a real `git worktree` (per `isolation: worktree`) only for genuine
  **same-file** concurrency that bed-ownership does not separate — not as the
  default for team parallelism.
- **Schedule longest-pole-first.** `charly eval run` has no bed-level concurrency and
  no `charly` cap — the limit is host CPU/RAM/podman. The lead runs ALL full beds as
  concurrent background tasks; order by expected DURATION, not bed count: launch
  the slow VM/desktop beds first and overlap the cheap pod beds, so wall-clock ≈
  the slowest single bed, not the sum.
- **Freeze `charly/*.go` during the bed phase.** `charly`'s stale-binary freshness guard
  gates every heavy verb the instant any `charly/*.go` is newer than `/usr/bin/charly`,
  so a teammate editing Go mid-bed-run aborts every other agent's next
  build/deploy/eval. For a SHARED-CORE (Go) cutover the lead lands the core
  first, runs ONE `task build:charly`, then fans out beds with Go frozen; a BED-LOCAL
  (YAML/candy/skills) cutover has no shared binary and needs no such barrier.

## B5 — PR path (no write access) + `gh` auto-approve

Detect permission: `gh repo view --json viewerPermission`
(ADMIN/MAINTAIN/WRITE → Mode 1, else Mode 2).

- **Mode 1 — write access:** the B1 direct path (ff-merge, tag, push, delete).
- **Mode 2 — fork + PR:** on R10 PASS, ensure a fork (`gh repo fork --remote`),
  push `feat/<slug>` to the fork, then
  `gh pr create --base main --head <fork>:feat/<slug>` with a body that pastes the
  R10 evidence and ends with `*Assisted-by: Claude (<tier>)*`. The PR is the
  deliverable; never force-push, never need upstream write.
- **`gh` auto-approve/merge (maintainer-side, with approve rights):** for an open
  PR, **fetch its head, review the diff, and run R10 against it**; ONLY on R10
  PASS (and only if the change ships its eval/test coverage) `gh pr review
  --approve` then `gh pr merge --rebase --delete-branch` (rebase keeps `main`
  linear, matching ff-only), then tag the new `main` HEAD. **Never a blind
  approve** — no approval/merge of unreviewed or R10-unverified code, whoever
  opened it. Required status checks / branch protection are respected, never
  bypassed, never force-merged.

## B6 — cross-repo R10 when a change is referenced via `@github`

The resolver (`EnsureRepoDownloaded`) fetches a producer repo from the REMOTE at
the pinned ref, so a producer change on a local `feat/` branch is invisible to a
consumer's R10 — **a local branch is not enough**. Staged landing (no
local-override):

1. Develop producer (A) + consumer (B) on the same `feat/<slug>`.
2. **Land the producer FIRST:** run A's own R10, ff-merge, **tag A `v<CalVer_A>`**,
   push — now an immutable, fetchable remote tag.
3. **Repoint the consumer:** `charly box reconcile` rewrites B's `@github.../A:...`
   pins to `v<CalVer_A>` (see `/charly-build:reconcile`).
4. **Authoritative consumer R10 against the real tag:** B's R10 now fetches A from
   the pushed `v<CalVer_A>` — verified against exactly what ships.
5. **Land the consumer** (ff-merge, tag B, push).
6. **New candy:** a new candy has no standalone R10 — its gate is the consuming
   image's build. A lands a **provisional** `v<CalVer_A>` (layer + `go test` /
   `charly box generate` smoke); step 4 (B's image R10 against that tag) is the real
   gate. On failure, fix A, land a **new** tag (immutable + accumulate — never
   move the old one), re-reconcile, re-run step 4.

Each repo gets ONE R10 against ITS final code — the producer against its own
change, the consumer against the producer's landed tag — and repos land
producer→consumer. Multi-level chains (A→B→C) recurse the same way.

## CalVer tag computation

`v<YYYY.DDD.HHMM>` from the current UTC push time. Every component is fixed-width
zero-padded (4-digit year, 3-digit day-of-year, 4-digit HHMM) so tags sort
chronologically under a plain alphanumeric sort:

```bash
git tag -a "$(date -u +v%Y.%j.%H%M)" -m "<subject>" HEAD
```

ONE fresh tag per push (a repo accumulates many), immutable (only ever added),
INDEPENDENT of `charly.yml` `version:` (the schema version, bumped only by a
`MigrationStep` raising `LatestSchemaVersion()`). A YAML schema/format change does
BOTH: raise `LatestSchemaVersion()` AND mint the tag. See `/charly-build:migrate`.

## After landing — cleanliness + report

- **Working-tree cleanliness.** After commit + landing, `git status` is clean
  in every repo. Untracked files that aren't part of the cutover (test
  artifacts, build outputs) belong in `.gitignore`; if they aren't, that's its
  own immediate-next cutover, not part of this one.
- **Report format.** The final message states: what was committed (commit
  subject + hash, per repo), the confidence tier with the proof that supports
  it, what was pushed, and the pasted R10 outputs (exploratory +
  fresh-rebuild). A worked commit message:

```
Fix: Add fuse-overlayfs for container startup

Tested via overlay session on LOCAL system.

Assisted-by: Claude (fully tested and validated)
```

## If R10 fails

R10 failure is a return-to-implementation signal, not a stopping point:

1. Run `/charly-internals:root-cause-analyzer` BEFORE attempting any fix —
   blind retry is FORBIDDEN.
2. Fix in the SAME working tree — never a follow-up PR.
3. Re-run the FULL R10 from a fresh `charly update`, not just the failing
   piece — a fix that survives only the targeted re-run is a regression in
   waiting.
4. Commit only when R10 passes end-to-end on the FINAL code.

## Cross-References

- CLAUDE.md "Post-Execution Policies" — the mandate this skill operationalizes.
- `/charly-internals:cutover-policy` — one-phase, atomic-commit, R10-at-the-end.
- `/charly-build:migrate` — `version:` ↔ tag coupling, per-push tags, push order.
- `/charly-build:reconcile` — cross-repo `@github` pin alignment used by B6.
- `/charly-eval:eval` — the eval-coverage gate (R10) every change must satisfy.
- `/charly-internals:root-cause-analyzer` — run on any R10 failure before re-trying.

## When to Use This Skill

Invoke before any `git` / `gh` action that commits, branches, pushes, merges,
tags, creates a PR, or approves/merges a PR — and whenever syncing to upstream or
pruning branches/worktrees across the main repo and its submodules.
