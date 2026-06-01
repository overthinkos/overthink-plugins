---
name: agents
description: |
  Claude Code multi-agent support in Overthink — sub-agents, dynamic workflows, and agent teams, and how each drives the existing `ov eval` disposable beds to test and verify. MUST be invoked before authoring or invoking an ov sub-agent / dynamic workflow / agent team, wiring agent-lifecycle hooks, or asking "which primitive should drive the R10 beds?".
---

# Agents, Workflows & Teams

Overthink is built to be driven from Claude Code's multi-agent primitives.
This skill is the authoritative reference for the three primitives, the ov
agent roster, the shipped workflows, the bed-scoped parallel-testing model for
teams, and the one rule that binds them all: **a bed run is R10-class — the
commit is gated on a full final-code bed test (pasted), but beds run freely
throughout to verify** (CLAUDE.md Law 5).

## The three primitives — when to use which

| | Sub-agent | Dynamic workflow | Agent team |
|---|---|---|---|
| What it is | A worker Claude spawns (Agent tool / @-mention) | A JS script the runtime executes | Multiple full Claude sessions: a lead + teammates |
| Holds the plan | Claude, turn by turn | The script | The lead + a shared task list |
| Intermediate results | Claude's context | Script variables | Each teammate's own context |
| Scale | a few per turn | dozens–hundreds of agents/run | 3–5 teammates |
| Lives in | `plugins/internals/agents/*.md` (or `.claude/agents/`) | `.claude/workflows/*.js` (run `/<name>`) | runtime only — `~/.claude/teams/`, NOT pre-authored |
| Reads CLAUDE.md | yes (full hierarchy, except Explore/Plan) | each `agent()` does | yes (each teammate) |

- **Sub-agent** — isolate a verbose side task (run a bed, probe a deploy) and
  get back a verdict; the noisy output stays out of the main context.
- **Dynamic workflow** — codify a repeatable fan-out (run every bed; audit
  every deploy config) as a script you can rerun. Triggered by the word
  `workflow` in a prompt, by `/effort ultracode`, or by a saved `/<name>`.
- **Agent team** — parallel *exploration/review* where teammates challenge
  each other (competing-hypotheses triage of an eval failure). Experimental;
  opt-in only (see "Agent teams" below).

**Preference (default): agents over background tasks — everything that CAN run as
an agent SHOULD run as an agent.** Prefer an addressable, operator-visible
**sub-agent** or **agent-team teammate** over an opaque background **dynamic
workflow**. **Team agents are the DEFAULT for parallel work** — the operator
watches and messages them live, which is exactly the visibility/control a
background workflow hides. Reach for a background `Workflow` only as a LAST
RESORT, when deterministic scripted control flow (loops / conditionals / large
fan-out) genuinely cannot be expressed as a team — and even then it surfaces its
work as agents and stays bed-scoped (see "Implementation workflows are bed-scoped
too"). Operator-facing agents beat opaque background tasks every time. **The one
exception is long-running work that outlives a single turn** (a VM/emulator eval
bed): no agent can reliably hold it — a sub-agent returns synchronously (its
background children die on return) and a teammate is torn down on idle — so it
runs as a harness-tracked background task owned by the persistent session, driven
by the completion notification (see "Handling a long-running bed — by mechanism"
under the binding rule). "Prefer agents" governs BOUNDED work.

## The ov agent roster (`plugins/internals/agents/`)

**Executors** — they RUN `ov eval` and return verbatim proof:

- **`eval-bed-runner`** — runs `ov eval run <bed>` (the full R10 sequence:
  build → eval image → deploy → eval live → fresh `ov update` → teardown) on
  a `kind: eval` disposable bed; returns per-step status, exit code (0 pass /
  1 infra / 2 checks-failed), and the failing-step log tail. The R10
  acceptance executor.
- **`deploy-verifier`** — read-mostly: `ov eval image` / `ov eval live` /
  `ov status` against an image or a running deploy (the ov repo's images OR a
  user's own deploy config). Answers "does this deploy config work?" without
  mutating anything.

**Enforcers** — they GATE claims (dev discipline):

- **`root-cause-analyzer`** — R1 mandatory invocation on any failure/anomaly;
  8-step RCA before any fix.
- **`testing-validator`** — blocks "it works" claims lacking the R10 proof;
  owns the 4-tier confidence table (must match CLAUDE.md).
- **`layer-validator`** — pre-edit `layer.yml` sanity gate; defers the full
  schema to `/ov-image:layer` + `ov image validate`.

Invoke by name in a prompt, `@`-mention, or the `Agent` tool (scoped id
`ov-internals:<name>`). Custom agents load at SESSION START, so the shipped
workflows do NOT depend on `agentType:` — they inline each agent's role in a
self-contained `agent()` prompt + `schema`, which runs even before a reload
registers a newly-added agent. Reach for `agentType:` only once the agent is
loaded (a fresh session) or when reusing the definition as an agent-team
teammate.

## The shipped workflows (`.claude/workflows/`)

- **`/verify-beds [bed …]`** — the commit-gating full-live-test fan-out, also
  usable for continuous verification throughout development. Runs each
  `kind: eval` bed (default: all) **in parallel** via `parallel()`, bounded by
  the runtime's 16-concurrent agent ceiling (KVM/libvirt are multi-tenant,
  podman builds distinct image tags concurrently), and aggregates pass/fail.
  Beds skipped for a missing host prereq are logged, never silently dropped.
- **`/audit-deploy-configs [image|deploy …]`** — validates + `ov eval image`
  + optional `ov eval live` + `deploy-verifier` over a set of deploy configs;
  aggregates a health report. Serves the "evaluate deployment configs, for AI
  and humans" goal.
- **`/triage-eval-failure <bed>`** — competing-hypotheses RCA of a failed bed
  run: parallel `root-cause-analyzer`-style agents each validate a hypothesis
  on the live bed, cross-check adversarially, converge on the root cause, and
  hand back a fix to re-run the real bed (per R1).

## Implementation workflows are bed-scoped too — never sequential codegen + review

The shipped workflows above VERIFY. A dynamic workflow that **implements** a
cutover (fans the coding out across `agent()` calls) obeys the SAME bed-scoped
discipline as an agent team — it is the workflow expression of the B3 model
(`/ov-internals:git-workflow`), not an exemption from it:

- **Partition the parallel work by `kind: eval` bed.** One disjoint disposable
  bed per parallel owner (`eval-pod` / `eval-k3s-vm` / `eval-local` /
  `eval-android-emulator-pod` / …). Disjoint beds = disjoint container/VM/image
  names + ports, so they run concurrently and safely.
- **Eval-test at EVERY stage, never only at the end.** Each parallel owner
  **verifies before it changes** (Risk Driven Development — proves its bed's
  high-risk assumptions, above all the composition, on its live bed/backend
  first, never trusting a doc or the code for a high-risk call) and runs its
  bed's real `ov eval run <bed>` as the fresh-rebuild R10.
- **Read-only review is an ADDITIONAL layer, NEVER a substitute.** A workflow
  that replaces real-deployment bed runs with a read-only diff-review phase is a
  protocol violation — the opposite of this skill and of CLAUDE.md "Agents,
  Workflows & Teams". Adversarial diff review is welcome ON TOP of the beds.
- **Compile-coherence is solved structurally, not by serializing.** A single Go
  package can't have N agents edit-and-build at once, but that is handled by
  shape, not by abandoning parallelism: the lead lands the **shared core first**
  (compile-clean), each parallel unit is an **independent `init()`-registered
  file** (no shared-file edits), and the one shared host **`ov` binary rebuild is
  a single barrier** between the parallel-implement and parallel-bed-R10 phases.
  Canonical shape: `Core (seq) → Implement (parallel by bed) → Integrate+build
  (seq barrier) → BedR10 (parallel by bed) → Review (parallel, read-only,
  optional)`.
- **Same binding rule** as below: disposable-only, commit-gated-not-run,
  no-scope-shrinking-flags, paste-proof survives delegation.

## The binding rule: running a bed is R10-class

`ov eval run <bed>` and `ov update` perform an unattended destroy + rebuild.
Therefore, for ANY agent or workflow that runs them:

- **Disposable-only (Law 4).** The sole authorization is the bed's explicit
  `disposable: true`. Agents run `kind: eval` beds, never arbitrary deploys.
- **The commit is gated, not the run (Law 5).** The git commit happens ONLY
  after a full live test of EVERYTHING — the FINAL code, on `disposable: true`
  beds — passes and is pasted. Running `/verify-beds`, `eval-bed-runner`, or
  any `ov eval run` THROUGHOUT development — in parallel or in the background,
  to validate assumptions before you change and to diagnose errors — is
  ENCOURAGED. A run that passes on an *intermediate* state simply does not
  authorize the commit; only the full final-code run does.
- **No scope-shrinking flags (Law 3.6).** Never pass `--no-rebuild` / `--keep`
  / `--on-*` / scenario filters unless the user named the flag this turn.
- **Paste-proof survives delegation.** Sub-agents are built to *summarize*,
  but R10 demands *pasted* proof. The executors return the raw verdict + exit
  code + failing-log tail; the **delegating (main) agent pastes it** into the
  conversation. A delegated bed run whose failure is summarized away is the
  exact fraud pattern the project bans.
- **Handling a long-running bed — by mechanism, not by who owns it.** A
  VM/emulator bed (`eval-k3s-vm`, `eval-android-emulator-pod`, the bootstrap-VM
  beds) runs for minutes-to-tens-of-minutes and its libvirt domain / emulator
  OUTLIVES a single turn. Run it by the mechanism, not a who-owns-it rule:
  - **Launch as a harness-tracked background task** (`run_in_background`). NEVER
    foreground — the Bash 120s/600s timeout kills the call mid-`vm-create`,
    orphaning the domain. NEVER a sleep/poll loop to "keep it alive" — that
    busy-poll is the exact R4 bandaid this replaces.
  - **The completion notification is the signal, not polling.** The harness
    re-invokes the LAUNCHING session with a `<task-notification>` when the run
    exits, so the launcher must SURVIVE to completion to receive it. The
    persistent main session does. An ephemeral sub-agent does NOT (the `Agent`
    tool returns synchronously — its background children die when it returns),
    and an idle teammate does NOT (its process tree is torn down on idle) — both
    orphan the bed. **Long beds belong to a session that lives to be notified;
    short beds (done within one turn / the 600s foreground budget) can be
    sub-agent- or teammate-owned.**
  - **Reconnect via durable state, never a held process handle.**
    `.eval/<bed>/<calver>/summary.yml` (overall `ok:` + per-step status) + the
    live domain/container ARE the source of truth: "done + verdict" =
    `summary.yml` exists; "still alive" = the `ov eval run` orchestrator is in
    the process table. On a suspected orphan — a `running` domain with NO live
    orchestrator — `ov vm destroy <entity>` (or remove the container) before
    re-running. You re-derive state from disk; you never "lose" a run.
  - **Paste-proof survives (Law 5).** The owner reports the verbatim
    `summary.yml` verdict + exit code; the lead pastes it.

## Hooks: lean pointers + deterministic gates (not walls of text)

Hooks in this project do TWO things and nothing more (see `.claude/hooks/`):

1. **Lean reminders** (`UserPromptSubmit`, `Stop`) that POINT to CLAUDE.md /
   skills — they never re-state R0–R10 (duplication drifts; CLAUDE.md is the
   single current source).
2. **Deterministic `PreToolUse` gates** (`pre-commit-gate.sh`,
   `pre-push-gate.sh`) that BLOCK (exit 2) only unambiguous, CLAUDE.md-stated
   invariants: `git commit --no-verify`, a missing/illegal `Assisted-by:
   Claude (<tier>)` trailer, the `theoretical suggestion` tier, and
   `git push --force` / `--force-with-lease`.

The honest division of labor: **hooks gate mechanical invariants; agents
judge proof.** Whether a tier is *justified* by the evidence is a reasoning
task — that stays with `testing-validator` + the pasted-proof rule, NOT a
regex in a hook. Never re-bloat the hooks back into CLAUDE.md copies.

## Agent teams (experimental — enabled in committed settings)

Agent teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) are **enabled in the
committed `.claude/settings.json`** (`env` block). The experimental caveats
remain: no in-process session resume (`/resume`/`/rewind` don't restore
teammates), one team at a time (clean up before creating another), no nested
teams (only the lead manages the team), and the lead is fixed for the team's
lifetime. Enabling requires a `claude` restart, because the `env` flag is read
at process start.

Teammates reuse the same agent definitions as roles — their `tools` + `model`
apply; the `skills`/`mcpServers` frontmatter does NOT on the team path (each
teammate loads `CLAUDE.md` + project/user skills on spawn, like any session).
Set the **Default teammate model** in `/config` (pick "Default (leader's
model)" to inherit). The `TaskCreated` / `TaskCompleted` / `TeammateIdle` hooks
can enforce gates (exit 2 = block + feedback); the shipped
`team-coordination-reminder.sh` is a soft pointer (exit 0).

### Bed-scoped parallel real-deployment testing

The **eval bed is the unit of ownership AND isolation** — it replaces the git
worktree. `validateEvalBeds` guarantees each `kind: eval` bed has a name
disjoint from every other deploy, `target ∈ {pod,vm,local}`, and
`disposable: true`; distinct beds therefore get distinct `ov-<bed>`
container / libvirt-domain / image names and disjoint ports, so they are
concurrent-safe. A bed pins an image → layers → files, so owning a bed owns
those source files.

Each bed is the teammate's **candybox** (CLAUDE.md "Candyboxing"): a disposable,
secured deployment stocked with the FULL `ov` + MCP + `ov eval` toolset, so the
owner can build / deploy / prove the real thing inside its boundary and rebuild
it fearlessly — never a tool-restricted sandbox.

The playbook:

1. **Lead partitions the beds** so no two teammates own the same bed.
2. **Each teammate is a bed-owner**: it runs its bed's full `ov eval run <bed>`
   (build → eval image → deploy → eval live → fresh `ov update` → teardown) on
   a real deployment via the `eval-bed-runner` role, and triages failures via
   `root-cause-analyzer`. Review/RCA are auxiliary — never a substitute for the
   live run.
3. **Verify before you change (Risk Driven Development)**: each teammate proves
   its bed's HIGH-RISK assumptions — above all the composition — on its standing
   bed BEFORE editing, never trusting a doc or the code for a high-risk call, so
   it is never disproven hours later.
4. **Default parallel.** Beds run concurrently, bounded by the runtime's
   documented 16-concurrent / 1000-total dynamic-workflow agent ceiling, which
   queues excess. KVM and libvirt are multi-tenant and podman builds distinct
   image tags concurrently, so pod and VM beds run alongside each other.
5. **The lead owns the single commit**, gated on the consolidated full
   final-code live test (the beds in parallel). Teammates never commit/push.

Worked partition (illustrative): A→`{eval-pod, eval-local}`,
B→`{eval-jupyter-pod, eval-versa-pod}`, C→`{eval-k3s-vm}` (VM, needs the
libvirt user session), D→`{eval-sway-browser-vnc-pod}` (heavy). All concurrent
→ multiple pods *and* a VM live at once; wall-clock ≈ the slowest chain, not
the sum.

## Cross-References

- `/ov-eval:eval` — the bed surface these agents/workflows drive (`ov eval
  run`/`image`/`live`, the `kind: eval` bed inventory, exit codes).
- `/ov-internals:disposable` — why `disposable: true` is the sole destroy
  authorization.
- `/ov-internals:git-workflow` — the R10-gated landing the executors feed.
- `/ov-internals:skills` — agent/skill discovery + the signpost convention.
- CLAUDE.md "Agents, Workflows & Teams" + Laws 4/5 + AI Attribution.

## When to Use This Skill

Invoke before authoring or invoking an ov sub-agent / dynamic workflow /
agent team, before wiring agent-lifecycle or commit/push gate hooks, and
whenever deciding which primitive should drive the `ov eval` beds for a given
verification.
