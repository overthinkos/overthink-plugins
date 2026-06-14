---
name: go-quality
description: |
  How to check charly's Go source for CLAUDE.md compliance + code quality and fix what's
  found. Covers golangci-lint v2 (the deterministic backbone) + the charly/.golangci.yml
  linter set, gopls, go vet, gofmt; the .go compliance checklist (R3 duplication, R4 ad-hoc
  workarounds, R5 dead/stale paths, repo invariants); the legitimate-pattern allowlist that
  kills false positives; triage + adversarial verification; and how to fix findings as a
  handful of large thematic R10-gated cutovers. MUST be invoked before any Go code-quality
  audit, golangci-lint run, dupl/duplication check, or CLAUDE.md-compliance sweep of charly/.
---

# go-quality — checking & fixing charly Go source for CLAUDE.md compliance

The `charly/` CLI is one large `package main` (~570 files, ~169K LOC). This skill is the
operational HOW for auditing it against the checkable subset of CLAUDE.md and driving the
fixes. It owns the *tooling + method*; the rule definitions live in `/charly-internals:strict-policy`
(R1–R5), the source map in `/charly-internals:go`, and landing in `/charly-internals:cutover-policy`
+ `/charly-internals:git-workflow`.

## Toolchain (all declared in the `dev-tools` candy)

| Tool | Role | Invocation (from `charly/`, where `go.mod` lives) |
|---|---|---|
| `golangci-lint` v2 | the deterministic backbone — bundles staticcheck/unused/errcheck/ineffassign/dupl/gocritic/misspell/revive/gocyclo/… | `golangci-lint run ./...` |
| `gopls` | references / call-sites / rename — confirm a symbol is truly unused before deleting | `gopls references`, `gopls call_hierarchy` |
| `go vet` | compiler-adjacent correctness | `go vet ./...` |
| `gofmt` | formatting drift | `gofmt -l .` |
| `go test ./...` | unit smoke (NOT the R10 gate) | `go test ./...` |

`golangci-lint` is **v2** — the config schema is `version: "2"` (not v1). `dupl` is a BUNDLED
linter, NOT a standalone tool (it is absent from the AUR — do not try to install it). There is
no `deadcode`/`staticcheck`/`misspell` standalone install either; all are bundled. No
`go install` of any linter (that would be an R4 workaround).

## The config — `charly/.golangci.yml`

Lives next to `charly/go.mod` (the module root), NOT the repo root. It enables the standard
set + the broadest curated quality/idiom/duplication linters, uncapped (`max-issues-per-linter:
0`) so an audit sees EVERY issue. **`gosec` is deliberately excluded**: on a subprocess-
orchestrating CLI its findings are dominated by the legitimate `exec.Command` pattern (the exact
false-positive class below); vulnerability scanning is `govulncheck`'s separate job.

## Running the audit

```bash
cd charly
golangci-lint run ./...                 # full broadest run (uncapped)
golangci-lint run ./... | grep -E '^\* '   # per-linter summary counts
gofmt -l .                              # files needing format
go vet ./...
```

Baseline at authoring (2026-06, broadest uncapped): **1017 issues** — errcheck 457,
staticcheck 242, unparam 100, unused 53, gocyclo 46, gocritic 43, errorlint 29, prealloc 29,
dupl 10, ineffassign 4, unconvert 3, misspell 1. Re-run to refresh; the per-linter shape is the
triage map.

## Auto-fix safety — NEVER blanket `--fix` (CRITICAL)

**`golangci-lint run --fix` CORRUPTS this source tree** (confirmed 2026-06-14, v2.12.2):
gocritic's autofixer rewrites multi-statement blocks into a one-liner whose body it emits as a
literal `{ ... }` elision placeholder — the `...` is gocritic's snip marker, written verbatim
into the file → `syntax error: unexpected ...` (broke `ssh_client.go` + `deploy_executor_nested.go`,
`go build` failed). Recovery: `git checkout -- charly/` then `go build ./...`.

- **`gofmt -w .` is the ONLY safe blanket auto-fix** (pure formatting, AST-identical).
- Fix every linter category **manually / per-finding with review**, or `--fix` scoped to a
  SINGLE proven-safe linter at a time — NEVER gocritic. golangci-lint is for *finding*, not
  auto-*fixing*. (Memory: [[golangci-lint-fix-corrupts-source]].)

## Dead-code (`unused`) — legitimate-keep patterns (do NOT delete)

golangci-lint `unused` is reliable for `package main`, but some flagged symbols are
intentionally retained — deleting them is wrong. Flag (don't delete) a symbol that:
- carries an explicit **retention doc-comment** ("kept for …", "retained because …");
- is referenced by a closure that is itself assigned-but-not-called (`check := func(){…}; _ = check`)
  — deleting the symbol breaks `go build` even though `unused` flags it;
- is named only in **comments / cross-references** (incl. comments in OTHER files / tests) —
  deleting orphans the reference;
- is deliberately-unwired scaffolding for a not-yet-shipped path.
Always confirm with `go build ./...` + `go test ./...` after deletions: a deletion that breaks
either means the symbol WAS used (transitively) — restore + flag it. A stale retention comment
whose named consumer no longer exists is its own R1 incident (fix the comment, then re-evaluate).

## The `.go` compliance checklist

**Linter-covered (deterministic — let golangci-lint find these, agents only triage + dedup):**
dead code/`unused`, `ineffassign`, `unconvert`, `staticcheck` correctness, `gocritic` idiom,
`misspell`, `gocyclo` complexity, `errcheck` unchecked errors, `errorlint` wrapping, `unparam`,
`prealloc`, and `dupl` cross-file/in-file duplication (R3).

**Agent-only (linters cannot express — grep + judgment):**
- **R4 workarounds**: `time.Sleep` poll/retry loops lacking a sync primitive; unnamed magic
  numbers/durations; retry-on-flake; environment-specific shims; ad-hoc
  `podman`/`docker`/`virsh`/`systemctl` that BYPASS charly (see allowlist for the legitimate ones).
- **R5 stale paths**: `deprecated`/`DEPRECATED`/`TODO` markers with un-swept old code paths;
  references to deleted identifiers (grep the identifier — only `CHANGELOG.md` should remain).
- **Repo invariants**: YAML-tag ↔ Go-identifier plural/singular symmetry; mode purity
  (`LoadConfig` must NOT read the deploy overlay / `charly.yml`); InstallPlan IR sync points;
  the "charly CLI is the only operational interface" rule.

## Legitimate-pattern allowlist — do NOT flag these (the #1 noise source)

- `exec.Command("podman"|"virsh"|"systemctl"|"pacman"…)` where **charly IS the orchestrator**
  — these are the operational interface, not ad-hoc bypasses. Forbidden is an ad-hoc such call
  against a charly-managed resource OUTSIDE the orchestration path.
- Idiomatic `errcheck` ignores: `defer x.Close()`, `fmt.Fprint*` to stderr/stdout, `os.Setenv`
  in tests — the fix is an explicit `_ = …` (makes intent visible), not handling.
- Named, justified `time.Sleep` polls in device/container/mount liveness (adb, gocryptfs) where
  no event/sync primitive is exposed by the underlying tool.
- Standard mode bits (`0o755`/`0o644`/`0o600`), well-known ports/IDs documented in domain code.

## Triage + adversarial verification (mandatory before any fix)

False positives auto-"fixed" become regressions. Every candidate finding is **adversarially
verified** — ≥3 skeptics, each prompted to REFUTE with a distinct lens (coincidence-not-dup /
legitimate-orchestration / no-sync-primitive-exists / idiomatic-ignore / would-break-surrounding-
idiom), **defaulting to refuted when uncertain**. Only majority-confirmed findings are fixed.

## Fixing — a handful of large thematic cutovers (NOT per-finding)

Group ALL confirmed findings into ~8 large cutovers, each fixing a whole homogeneous category
(tens-to-hundreds) as ONE atomic commit, R10-gated as a whole. Order low-risk/high-value →
higher-churn: dead-code → dupl(R3) → staticcheck → errcheck → idiom/quality → gocyclo →
agent-only rules. Cutovers land SEQUENTIALLY (one Go package — no concurrent mega-edits); the
EDITS within a cutover parallelize by file-cluster.

## R10 gate by change class (the commit gate)

`go test ./...` + `golangci-lint run` + `task build:charly` are SMOKE, not the gate. The gate
is `charly check run <bed>` on the bed that EXERCISES the change (cross-cutting tree-wide
categories → `--all-beds`), disposable-only, fresh-rebuild, zero warnings, pasted proof. See
`/charly-check:check` "R10 gate by change class" and `/charly-internals:cutover-policy`.

## Cross-References

- `/charly-internals:strict-policy` — R1–R5, RDD, the forbidden-pattern catalogs (the WHAT).
- `/charly-internals:go` — the Go source map (file → responsibility).
- `/charly-internals:skills` — the companion **skill↔code source-map sync audit** (docs side).
- `/charly-internals:cutover-policy` + `/charly-internals:git-workflow` — landing the fixes.
- `/charly-check:check` — the R10 bed gate every code cutover passes.

## When to Use This Skill

Invoke before running golangci-lint / dupl / a duplication check, auditing charly Go source
for CLAUDE.md compliance or code quality, or planning/landing a compliance fix sweep.
