---
name: agents
description: |
  Claude Code multi-agent support in Overthink — sub-agents, dynamic workflows, and agent teams, and how each drives the existing `ov eval` disposable beds to test and verify. MUST be invoked before authoring or invoking an ov sub-agent / dynamic workflow / agent team, wiring agent-lifecycle hooks, or asking "which primitive should drive the R10 beds?".
---

# Agents, Workflows & Teams

Overthink is built to be driven from Claude Code's multi-agent primitives.
This skill is the authoritative reference for the three primitives, the ov
agent roster, the two shipped workflows, and the one rule that binds them
all: **anything that runs an `ov eval` bed is R10-class** (CLAUDE.md Law 5).

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

- **`/verify-beds [bed …]`** — the R10 fan-out. Runs each `kind: eval` bed
  (default: all) and aggregates pass/fail. **Resource-aware:** light
  pod/local beds pipeline; the VM/KVM beds (`eval-k3s-vm`,
  `eval-android-emulator-pod`) run sequentially to avoid libvirt/`/dev/kvm`
  contention; beds skipped for a missing host prereq are logged, never
  silently dropped.
- **`/audit-deploy-configs [image|deploy …]`** — validates + `ov eval image`
  + optional `ov eval live` + `deploy-verifier` over a set of deploy configs;
  aggregates a health report. Serves the "evaluate deployment configs, for AI
  and humans" goal.

A third **competing-hypotheses triage** pattern (parallel
`root-cause-analyzer`-style agents converging on an eval failure, per R1) is
a recommended composition — author it as a workflow if/when needed.

## The binding rule: running a bed is R10-class

`ov eval run <bed>` and `ov update` perform an unattended destroy + rebuild.
Therefore, for ANY agent or workflow that runs them:

- **Disposable-only (Law 4).** The sole authorization is the bed's explicit
  `disposable: true`. Agents run `kind: eval` beds, never arbitrary deploys.
- **R10 is the last step, never a parallel/background track (Law 5).** Do NOT
  launch `/verify-beds`, `eval-bed-runner`, or any `ov eval run` while
  implementation tasks are still open. It is the dedicated verification phase
  AFTER the cutover's code is complete.
- **No scope-shrinking flags (Law 3.6).** Never pass `--no-rebuild` / `--keep`
  / `--on-*` / scenario filters unless the user named the flag this turn.
- **Paste-proof survives delegation.** Sub-agents are built to *summarize*,
  but R10 demands *pasted* proof. The executors return the raw verdict + exit
  code + failing-log tail; the **delegating (main) agent pastes it** into the
  conversation. A delegated bed run whose failure is summarized away is the
  exact fraud pattern the project bans.

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

## Agent teams (experimental — opt-in only)

Agent teams (`CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`) reuse the same agent
definitions as teammates (their `tools` + `model` apply; `skills`/`mcpServers`
frontmatter does NOT on the team path). With teams on, the `TaskCreated` /
`TaskCompleted` / `TeammateIdle` hooks can enforce R10/skills-first as
quality gates (exit 2 = block + feedback). Not enabled in committed settings
(experimental: no session resume, one team at a time, no nested teams) — turn
it on per-session if you want parallel review/triage.

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
