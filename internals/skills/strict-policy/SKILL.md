---
name: strict-policy
description: |
  Operationalization of CLAUDE.md R1-R5 — the engineering-discipline rules
  that come BEFORE runtime verification. Covers: (R1) RCA on every failure
  via /ov-internals:root-cause-analyzer; (R2) no "pre-existing" / "out of scope" /
  "follow-up PR" classifications; (R3) no code duplication, generic over
  ad-hoc; (R4) no ad-hoc workarounds; (R5) hard cutover deletes the
  deprecated path AND every stale reference in the same commit.
  MUST be invoked when a failure / warning / anomaly surfaces, when the
  same pattern is about to land in a second surface, when a sleep / retry /
  magic-number is tempting, or when a cutover commit is about to ship.
---

# strict-policy — operationalizing R1–R5

## Overview

R1–R5 in CLAUDE.md are the **engineering-discipline gate**. They sit ABOVE the runtime-verification rules R6–R9 and ABOVE the final acceptance gate R10. The order is deliberate: discipline failures are what produce the runtime failures that R6–R9 catch — so the rules that prevent discipline failures must come first.

This skill is the operational reference for R1–R5. Each section below restates the rule, lists what it forbids precisely, lists what it permits, gives a worked example drawn from this project's commit history, and notes how it interacts with the other rules. The cautionary tales are deliberately drawn from recent commits in the same session — the rules exist because these specific failure modes happened, not because they are theoretical risks.

A violation of any R1–R5 rule (or any of R6–R10, or the "Prioritize Clean Architecture Above All Else" section in CLAUDE.md) FORBIDS commit. There is no "downgrade tier and ship anyway" path. The agent fixes the violation in the same working tree and re-runs all verification, OR escalates to the operator and STOPS. No commit ships at any tier with a known violation. See CLAUDE.md "AI Attribution" section.

## R1. RCA on every failure — no transient-flake classification

**The rule.** Every failure, error, anomaly, or warning surfaced by ANY tool (build, test, validator, runtime, eval, deploy, lint, hook) triggers IMMEDIATE invocation of `/ov-internals:root-cause-analyzer` BEFORE any remediation attempt. The first occurrence is the investigation trigger; there is no second-occurrence threshold.

**Forbidden first responses.** "probably a flake" / "rerun and see" / "transient" / "intermittent" / "works on retry" / "environmental" / "let me try once more" / "maybe the network was slow" / "let me clear the cache and re-run". These are confessions, not defences. Each is a way of avoiding the analysis the rule mandates.

**What is permitted.** The `/ov-internals:root-cause-analyzer` agent's 8-step process. After running it, if the analyzer concludes the root cause is genuinely external (network partition, upstream registry outage, kernel bug with a tracked upstream report), the conclusion is documented in the conversation **with evidence** — never assumed. Only after that documented conclusion is "retry" an authorized response.

**Why it matters.** Failures that look transient often aren't. A test that fails 1-in-N times because of a race condition will fail 1-in-(N/scale) times under load — pretending the failure is "flaky" hides the race instead of fixing it. R1 forces the investigation early, when the symptoms are simple, before the bug accumulates obscuring complications.

**Interaction with other rules.** R1 is the first response to ANY failure surfaced by any other rule's verification step. R7 (mandatory end-to-end gate) produces failures that R1 must investigate. R8 (generated-artifact invariants) produces failures that R1 must investigate. R10 (disposable + fresh-rebuild) produces failures that R1 must investigate. R1 is also the first response to a self-detected anomaly mid-session — including during planning, exploration, or normal coding.

## R2. No "pre-existing" / "out of scope" / "unrelated" / "follow-up PR" classifications

**The rule.** Every issue surfaced during the active cutover — failing test, validator warning, runtime crash, deprecated-marker hit, dead-code reference, stale doc paragraph — is fixed in the SAME working tree as the cutover, OR escalated to the operator for explicit re-scoping. The classifications "pre-existing", "unrelated to this change", "out of scope", "follow-up PR", "tracked separately", "we'll get to it later" are FORBIDDEN.

**Forbidden phrasings.** "this was already failing before my change" / "unrelated to this PR" / "out of scope for this cutover" / "I'll file a follow-up issue" / "this is tracked separately" / "we can address that later" / "noted but not required for this commit" / "intentionally deferred to keep the diff focused".

**What is permitted.** Two paths:

1. **Fix in the same working tree.** Same commit as the active cutover; same atomic change. The fix lands with the rest of the cutover.
2. **Escalate to the operator** with an explicit ask: "I encountered $X during this cutover. Per R2 I cannot defer it. Should I (a) fix it now in this same commit, (b) abort the active cutover and address $X first, or (c) explicitly re-scope?" The operator's response authorizes the path; silence does not.

**Cautionary tale.** Commit `8a275e8` (cachyos-dx rename) deferred `TestRenderTaskCommandMkdir` as "pre-existing, unrelated". The fix actually landed in the next commit `22b5d0d` — meaning the test was failing in production for the duration between those two commits. Per R2, the fix should have been part of `8a275e8`. The cost of deferring was a window of broken tests on `main`. The cost of fixing-in-place would have been ~5 lines of code and 30 seconds of attention. R2 closes this escape hatch absolutely.

**Why it matters.** "Pre-existing" is the most common AI-agent escape hatch — it lets the agent claim the work is complete while leaving brokenness in the tree. Every "pre-existing" deferral compounds: the next cutover finds two pre-existing issues, the third finds three, and the codebase entropy grows monotonically. R2 forces the agent to either pay the small fix-cost now OR escalate to a human; it removes the silent third option of "leave it".

**Interaction with other rules.** R2 extends old-R6 ("I'll fix it in Phase 2 is not in the approved plan unless the plan says so"). Old-R6 covered approved-plan phasing; new-R2 extends to incidentally-surfaced issues mid-session. Together they say: **no deferral, neither planned nor incidental.**

## R3. No code duplication; generic, reusable solutions over ad-hoc patches

**The rule.** On the FIRST surface where the same pattern, predicate, filter, transform, or guard appears in two places, refactor to ONE shared abstraction in the SAME working tree. Every fix MUST apply cleanly to ALL surfaces it logically covers, not just the surface that prompted the report.

**Forbidden patterns.** Sibling-layer naming (`<name>-host`, `<name>-pod`, `<name>-bootc` when they share content); parallel filter functions in adjacent files; per-call-site re-implementations of the same predicate; copy-pasted YAML stanzas across multiple layer.yml files; copy-pasted Go function bodies with one-token differences; "let me just patch this one consumer for now and unify later"; "the abstraction is unclear, so I'll duplicate for now and refactor when the pattern firms up".

**What is permitted.** ONE shared abstraction, in the obvious shared location. If the abstraction is unclear, the answer is to think harder, ask the operator, or escalate — not duplicate. If the cost of the refactor is high enough to warrant a discussion, escalate; do not silently duplicate.

**Cautionary tale.** The `*-host` sibling-layer pattern (`virtualization`/`virtualization-host`, `ov-full`/`ov-full-host`) accumulated for months because no rule banned the duplication on day one. Each time a new host-vs-container difference came up, the response was "spawn a sibling layer" instead of "extend the existing layer with init-system-aware logic". By 2026-05 the duplication had crystallized into divergent surfaces that drifted in their package lists, eval probes, and service definitions. Commit `22b5d0d` deleted the entire `*-host` namespace and replaced it with the unified `service:` schema mixed-entry pattern — but the deletion would not have been necessary if R3 had been in force from day one.

**Worked example.** Commit `22b5d0d` collapsed three previously-divergent service-filter paths — `LocalDeployTarget` silent-skip, `VmDeployTarget` lazy render, `OCITarget` per-entry filter — into ONE compile-time filter in `compileServiceSteps`. The first attempt at the polymorphism fix added a band-aid in just `LocalDeployTarget`; the operator caught it and demanded the canonical compile-time filter. That demand IS R3 — applied retroactively because R3 was not yet codified. With R3 in place, the band-aid attempt would have been forbidden from the start.

**Why it matters.** Duplication has compounding cost. Two divergent copies become three, then four, then eight; each copy hides bugs the others fixed. The cost of the unification grows superlinearly with the number of copies. R3 enforces unification at copy-count = 2 — the cheapest possible moment.

**Interaction with other rules.** R3 is paired with the architectural-philosophy framing in CLAUDE.md "Prioritize Clean Architecture Above All Else" — which now contains three labeled sub-paragraphs (No duplication on first surface, Generic over ad-hoc, No workarounds) that mirror R3 + R4 from the architectural angle. Both framings are binding.

## R4. No ad-hoc workarounds — sleep loops, retry-on-flake, magic-number tuning, "works on my machine" fixes are FORBIDDEN

**The rule.** Sleep loops, retry-on-flake harnesses, magic-number tuning, hardcoded paths chosen because "the standard one was busy", environment-specific shims, and "works on my machine" fixes are FORBIDDEN. Every fix must apply cleanly across every supported environment.

**Forbidden patterns.** `sleep 5; retry` (race-condition cover-up); `for i in 1..3 do try; done` (retry loops disguising flake); hardcoded ports chosen because "8080 was busy" (port-allocation bug disguised as a config); environment-specific paths like `/Users/$USER/...` in shipped code; default-fallbacks that hide a missing config (silent fallback to a wrong value); "this is what worked when I tried it locally" (single-environment validation).

**What is permitted instead.**

| Forbidden pattern | Authorized replacement |
|---|---|
| `sleep N; check` | Synchronization primitive: file lock, readiness probe, condition variable, deterministic ordering |
| Retry on flake | Identify the race; fix the race; remove the retry |
| Magic number | Named constant, sourced from config, validated on load |
| Hardcoded port | Port allocation from a registry, or service-discovery lookup |
| Environment-specific path | Standard XDG/FHS path resolved at startup |
| "Works on my machine" fix | Cross-environment validation before the fix ships |

**Cautionary tale.** None in this session yet — the rule is preventive. R4 exists to forbid the patterns BEFORE they crystallize. Each forbidden pattern is the kind of "quick fix" that, once accepted, becomes tribal knowledge ("oh, that test always needs a sleep; that's just how it is"). R4 closes the door before the tribe forms.

**Why it matters.** "Temporary" fixes never get removed. Every sleep loop in the codebase was added with a "this is just for now" justification that was never revisited. R4 forbids the pattern at addition time, before the temporary becomes permanent.

**Interaction with other rules.** R4 is paired with R3 in the architectural-philosophy sub-paragraph "No workarounds" in the "Prioritize Clean Architecture" section. R4 violations also typically violate R1 — the workaround is an attempt to dodge the failure rather than RCA it.

## R5. Hard cutover: deprecated path AND every stale reference deleted in the same change

**The rule.** When a cutover introduces a replacement, the SAME commit deletes (a) the deprecated code path, (b) every comment / TODO / DEPRECATED marker referencing the old path, AND (c) every reference, comment, docstring, error message, skill paragraph, migration help-text, test fixture, or hook string naming a deleted identifier. After commit, `git grep '<deleted-id>'` returns ONLY historical mentions in changelog / history-note / migration-help-text contexts.

**The acceptance test.** A cutover is not "clean" until you can run `git grep '<deleted-id>'` and have ZERO live (non-historical) hits. Historical hits in changelog entries, history-note paragraphs, migration command help-text, and explicit "renamed from X" / "previously called X" cautionary-tale text are permitted — these intentionally preserve the old name to help users migrate. Everything else is a violation.

**Cautionary tales.**

1. **`image.yml` silent-skip regression.** The unified-yaml cutover deleted `image.yml` from the schema but the new `overthink.yml` path silently skipped a build stage that the old `image.yml` path had wired. The test of the cutover was supposed to be "rebuild from the new config, run the resulting image, observe the service reach steady-state" — but because the build skipped a stage, the resulting image was missing the init-system config. R5 specifies "verify zero stale references via the grep self-test" precisely to catch this class of regression.

2. **Stale references in this session.** Both `8a275e8` and `22b5d0d` had to do significant grep-and-fix work on stale references because the prior schema didn't enforce R5's same-commit-sweep requirement. Commit `8a275e8` introduced `cachyos-dx` and had to delete every reference to `qc`. Commit `22b5d0d` introduced `ov-cachyos` and had to delete every reference to `cachyos-dx` AND `qc` (since both retired in the same window). Each cutover ran a `git grep` self-test before commit; R5 codifies this self-test as a non-negotiable acceptance criterion.

**What is permitted in historical contexts.** Changelogs, history-note paragraphs in CLAUDE.md, migration-command help-text that names the legacy form for the user's benefit ("rename `qc` to `ov-cachyos`"), cautionary-tale paragraphs in skills that name the legacy form for instructional purposes. The grep self-test should distinguish these via context.

**Why it matters.** Stale references confuse new contributors and AI agents. A code search for `qc` that returns matches in `deploy.yml` suggests the deployment is still live; a search that returns matches only in `CHANGELOG.md` suggests it was retired. R5's grep self-test enforces this distinction.

**Interaction with other rules.** R5 extends old-R5 ("Never claim the cutover is clean until the old artifact is gone AND the new one runs") to cover stale references everywhere, not just the deleted artifact itself. R5 is the cleanup discipline that R3 enables — once you've refactored to the unified abstraction, R5 ensures every old reference points to the new one.

## How R1–R5 interact with R6–R10

R1–R5 are **authoring discipline** — what you do during the work.

R6–R9 are **artifact discipline** — what the produced artifact must be.

R10 is **live-system discipline** — what the deployed-and-running system must do.

The three layers compose. A cutover that violates R3 (duplication) but passes R7 (end-to-end gate) is still a violation — the duplication is a future bug. A cutover that passes R3 but fails R10 (fresh-rebuild verification) is still a violation — the artifact failed live. The three layers are AND-gated; **all** must pass.

Per CLAUDE.md "AI Attribution" section: a violation at any layer FORBIDS commit. The four-tier table describes the proof level the agent has when committing IS permitted; a known violation means committing is NOT permitted, regardless of tier. The agent fixes the violation or escalates to the operator — never both downgrade and ship.

## Cross-references

- `/ov-internals:cutover-policy` — the operationalization of R5 for schema/API/rename changes specifically. Cutover-policy is one of strict-policy's children — R5 governs all hard-cutover behavior, and cutover-policy operationalizes it for the most common case.
- `/ov-internals:root-cause-analyzer` agent — the R1 mandatory-invocation target. The agent's 8-step process is the only authorized first response to a failure.
- `/ov-internals:disposable` — R10's verification target. Strict-policy R5 (cutover) cooperates with R10 (verification) to ensure the post-rename state is both clean and live.
- `/ov-internals:skills` — the meta-skill for skill maintenance. R5's stale-reference sweep includes skill paragraphs; the skills meta-skill has a "When to Update Skills" row dedicated to R5 self-test failures.
- CLAUDE.md "Ground Truth Rules" — the canonical source for R1–R10. Strict-policy operationalizes R1–R5 specifically.
- CLAUDE.md "Prioritize Clean Architecture Above All Else" — the architectural-philosophy framing of R3 + R4. Both framings (procedural rules R3+R4 AND architectural-philosophy section) are binding.
- CLAUDE.md "AI Attribution" — the no-commit-on-violation clause that gives R1–R5 their teeth. Any violation FORBIDS commit. No tier downgrade. No "ship at lower tier". Fix or escalate.

## When to Use This Skill

**MUST be invoked** when:

- A failure / error / anomaly / warning surfaces from any tool, at any time during a session. R1's RCA mandate fires immediately.
- A pattern, predicate, or filter is about to land in a second place. R3's first-surface refactor mandate fires.
- A sleep, retry, magic number, or environment-specific shim is tempting. R4's forbidden-patterns list fires.
- A cutover commit is about to ship. R5's grep self-test mandate fires.
- An issue surfaces mid-session that "might be pre-existing" or "unrelated to my change". R2's no-deferral mandate fires.

The skill is also a useful read **before** starting a non-trivial cutover — both as a refresher and as a way to internalize the forbidden-internal-voice triggers in each rule's "Forbidden patterns" / "Forbidden phrasings" lists.
