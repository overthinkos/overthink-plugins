# harness — Drive AI agents through BDD-scenario iteration cycles

## Overview

`ov harness` is the project's iterative AI driver. Same loop, two modes:

- **Benchmarking** — score an AI against pending BDD scenarios; iterate
  until plateau.
- **BDD development** — write failing scenarios first, let the AI drive
  code that makes them pass.

Both modes share the machinery: per-iteration git clone + AI-CLI invocation +
nested `ov image build` + `ov image test --format yaml` scoring +
plateau-bounded loop. Only the prompt differs.

This is the **2026-04-26 successor** to `ov benchmark`. `harness.yml` carries
THREE kind-entities side by side:

- `kind: ai`     — reusable AI-CLI catalog (unchanged).
- `kind: recipe` — **pure spec**: description + BDD scenarios. Every scenario
  declares `pod: <name>` — the one container its steps probe.
- `kind: score`  — **runner config**: target (`pod`/`vm`/`host`), AI list,
  plateau, prompt, MCP endpoint, notes, env, plus `recipes:` — the ordered
  list of recipes it evaluates against in every iteration.

A score with `recipes: [a]` is a single-recipe run. A score with
`recipes: [a, b, c]` is a multi-recipe set: every iteration the AI is
shown all three recipes via `${RECIPES}` and the harness scores against
the union (sum of solved scenarios across a, b, c).

## Three-actor split — hard-edged

**HARNESS** (host-side `bin/ov harness run`):
- For pod targets with `disposable: true`: rebuilds the bench-pod fresh
  per run via `ov rebuild --reuse-image <pod>` BEFORE dispatching.
  The bench-pod is the SOLE disposable resource the harness manages.
- Dispatches the iteration loop into the bench-pod via `podman exec`.
- After each iteration: probes every scenario in every recipe at
  `ov-<scenario.pod>` via `podman exec` (against the bench-pod's
  nested podman socket).

**AI** (claude inside bench-pod):
- Reads scenarios via `${RECIPES}` / `${SCENARIOS}` / `ov harness scope`.
  Each scenario's rendered YAML shows `pod: <name>` — the AI reads
  that to know which containers to spawn.
- Does whatever is needed: `ov image build`, `ov deploy add` (each
  pod a scenario references), edit files, install packages. Full
  sudo + nested podman.
- Self-evaluates via `bin/ov test <pod>` before exiting each iteration —
  same scenarios the harness will score.

**RECIPE** (the spec author):
- Pure declarative scenarios. Each carries `pod: <name>` — the one
  container its steps probe. Multi-pod recipes are just scenarios
  with different `pod:` values in the same recipe.

## ONE primitive: `scenario.pod`

Every scenario declares its scoring target via ONE field at the
scenario level:

```yaml
scenario:
  - name: <scenario-name>
    pod: <container-name>      # REQUIRED — all steps probe ov-<container-name>
    steps: [...]
```

The harness scoring code does `containerName := "ov-" + scenario.Pod`
and dispatches every step in that scenario through `podman exec
ov-<pod>`. No defaults, no fallbacks, no cascade.

Validator: every scenario inside a `recipe:` block MUST set `pod:`.
Hard-error at load time otherwise (with hint pointing at the offending
recipe + scenario name).

`score.target_image` and `score.deployment` are KEPT as **purely
informational** prompt-token hints (`${TARGET_IMAGE}`, `${DEPLOYMENT}`).
Scoring NEVER consults them.

The loop is **plateau-bounded — there is no max-iteration ceiling.**
As long as the AI's score keeps improving, the run continues; it exits
once the plateau counter reaches `score.plateau_iteration`. The prompt
surfaces `${SCORE_DELTA}` (improvement vs previous iteration) and
`${ATTEMPTS_LEFT}` (`plateau_iteration - plateau_counter`) so the AI
knows where it stands.

## Quick Reference

| Command | Purpose |
|---|---|
| `ov harness run <score>` | Drive an AI through iteration cycles for the named score |
| `ov harness sync-credential <score>` | Copy AI credentials into the score's target |
| `ov harness list-ai` | Show configured AIs (with version_command + credentials count) |
| `ov harness list-recipe` | Show configured recipes (pure spec) |
| `ov harness list-score` | Show configured scores (runner config + recipes list) |
| `ov harness list` | List past runs under `.harness/<score>/` |
| `ov harness report [<score>] [<calver>]` | Print a past `result-<calver>.yml` |
| `ov harness scope` | **AI-facing**: print current iteration scope |
| `ov harness last-test-tag` | **AI-facing**: prior iteration's image tag |
| `ov harness note read [<score>]` | Print persistent NOTES.md memory |
| `ov harness note append <score> <text>` | Atomic timestamped note append |
| `ov migrate harness` | One-shot legacy → kind-split migrator |

## `harness.yml` shape — `ai:` + `recipe:` + `score:`

```yaml
ai:                                     # kind: ai — reusable AI-CLI catalog
  claude:
    description: { feature: "Anthropic Claude Code CLI" }
    command: [claude, -p, --dangerously-skip-permissions, "${PROMPT}"]
    version_command: [claude, --version]
    credential:
      - {src: ~/.claude/.credentials.json, dst: ~/.claude/.credentials.json}

recipe:                                 # kind: recipe — pure spec
  tier1-easy:
    description: { feature: "Tier 1 — marker file." }
    scenario:
      - name: marker-file-exists
        pod: harness-tier1              # REQUIRED — scoring target
        steps:
          - then: "/etc/ov-bench-marker exists"
            file: /etc/ov-bench-marker
            exists: true

  tier4-redis:
    description: { feature: "Tier 4 — redis + redis-client cross-pod." }
    scenario:
      - name: redis-server-running
        pod: redis                      # this scenario probes the redis pod
        steps:
          - then: "redis-server is running"
            process: redis-server
            running: true
      - name: redis-client-has-cli
        pod: redis-client               # this scenario probes the redis-client pod
        steps:
          - then: "redis-cli is installed"
            command: "command -v redis-cli"
            exit_code: 0

score:                                  # kind: score — runner config
  default:
    description: { feature: "Default benchmark." }
    pod: bench-pod                      # bench-pod = sandboxed iteration host
    ai: [claude]
    plateau_iteration: 3                # the only loop bound
    target_image: "bench-target"        # informational ${TARGET_IMAGE}
    deployment: "harness-default"       # informational ${DEPLOYMENT}
    notes: true
    mcp_endpoint: "http://localhost:18765/mcp"
    env: {}
    recipes: [tier1-easy, tier4-redis]
    prompt: |
      ...
```

The bench-pod entry (`bench-pod` above) lives in the host's
`~/.config/ov/deploy.yml` with `disposable: true` and a project
bind-mount; the harness rebuilds it fresh per run.

## Recipe vs Score — what goes where

| Concern | Lives on `recipe:` | Lives on `score:` |
|---|---|---|
| Scenarios (BDD) | YES | NO |
| `scenario.pod:` (REQUIRED on every scenario) | YES | n/a |
| `description:` | YES (the spec's narrative) | YES (the run's narrative) |
| Target — `host:`/`pod:`/`vm:` | NO | YES (xor: exactly one) |
| `disposable:` (on the deploy entry, not the score) | NO | NO (lives in deploy.yml) |
| `ai:` list | NO | YES |
| `plateau_iteration:` | NO | YES |
| `tag:` (Gherkin filter) | NO | YES |
| `target_image:`, `deployment:` (INFORMATIONAL prompt tokens) | NO | YES (optional) |
| `prompt:` template | NO | YES |
| `notes:` toggle | NO | YES |
| `mcp_endpoint:`, `env:` | NO | YES |
| `recipes: [...]` | NO | YES |

Recipes are reusable spec atoms. Scores are how you actually run them. A
score's `recipes:` list is ordered; the harness preserves that order
when concatenating scenarios for scoring and when rendering `${RECIPES}`.
Repetition in `recipes:` is rejected at validation. Empty `recipes:` is
rejected. Unresolved names are rejected.

## Where the harness runs — `pod:` / `vm:` / `host:` (exactly one, on the score)

| Field | Value | Meaning |
|---|---|---|
| `pod: <name>` | name of a running pod deployment | Run iterations inside this pod via `podman exec` |
| `vm: <name>` | name of a VM | Run iterations inside this VM via SSH |
| `host: true` | (boolean) | Run iterations on the local machine directly |

`host: true` requires `disposable: true` on the score — host iterations
destructively edit `$PWD`'s git tree on a per-run branch.

## Filesystem layout under `.harness/`

```
.harness/
└── <score-name>/                       # was <recipe-name> pre-cutover
    ├── note/
    │   └── <run-id>.md                 # per-run notes (notes: true)
    ├── results/
    │   └── result-<calver>.yml         # one per ov harness run
    └── runs/
        └── <run-id>/
            ├── repo/                   # per-run git clone
            └── iter1/
                ├── runner.log
                ├── build.log
                ├── prompt.md
                ├── prompt-arg.md
                ├── scope.yml
                ├── test-output.yaml
                └── score.yml
```

### Result file body

```yaml
schema: 1
score: default                          # was `recipe:` pre-cutover
recipes: [tier1-easy, tier2-medium, tier3-hard]   # NEW
calver: 2026.115.1430
run_id: 20260425-143000-abc123
ai: claude
where: { kind: host }
target_image: bench-target
plateau_iteration: 3
mcp_endpoint: "http://localhost:18765/mcp"
exit_reason: plateau                    # plateau | solved-all | dry-run | interrupted
final_score: 4
best_score: 4
ovharness_branch: ovharness/...
iteration:
  - k: 1
    score: 1
    score_delta: 1                      # NEW — improvement vs previous
    plateau_counter_after: 0
    solved_id: [recipe:tier1-easy:0]
```

## Memory — persistent notes across runs

Each score gets `.harness/<score>/note/<run-id>.md`. The harness exposes:

| Verb | Behavior |
|---|---|
| `ov harness note read [<score>]` | Print the file. Defaults to `$OV_HARNESS_SCORE` env (set during iterations). |
| `ov harness note append <score> <text>` | `O_APPEND` write with header `## <calver> run=<run-id> iter=<k> ai=<name>`. Requires `OV_HARNESS_NOTES_FILE` (auto-set inside iterations). |

Disable per-score with `notes: false` — `${NOTES}` then expands to empty.

## Substitution tokens

| Well-known token | Source |
|---|---|
| `${PROMPT}`, `${PROMPT_FILE}` | Rendered prompt text / file path |
| `${WORKSPACE}` | Per-run repo dir |
| `${RUN_ID}`, `${SCORE_NAME}`, `${AI_NAME}` | Run identity |
| `${TARGET_KIND}`, `${TARGET_NAME}` | "pod" / "vm" / "host" + name |
| `${TARGET_IMAGE}`, `${TAG}` | Score inputs |
| `${ITERATION}` | Current iteration k (1-based) |
| `${PLATEAU_ITERATION}`, `${PLATEAU_COUNTER}` | Plateau bookkeeping |
| `${ATTEMPTS_LEFT}` | `plateau_iteration - plateau_counter` |
| `${BEST_SCORE}` | Highest score seen so far |
| `${SCORE_DELTA}` | Score change since previous iteration |
| `${MCP_ENDPOINT}` | Score's `mcp_endpoint:` (default `http://localhost:18765/mcp`) |
| `${NOTES}` | Per-iteration NOTES.md snapshot |
| `${SCENARIOS}` | Flat union of in-scope recipes' scenarios (per-phase when `progressive: true`) |
| `${RECIPES}` | Per-recipe-grouped YAML block (one entry per in-scope recipe) |
| `${PHASE}` | Current phase number (1-indexed; 0 when non-progressive) |
| `${PHASE_TOTAL}` | Total number of phases (== `len(score.recipes)` when progressive) |
| `${PHASE_RECIPES}` | Comma-joined in-scope recipe names for this phase |
| `${PHASE_INTRO}` | Pre-rendered "Phase N of M — added recipe: ..." preamble line |
| `${DEPLOYMENT}` | Score's `deployment:` |

**Substitution precedence**: well-known token → `score.env[KEY]` →
`ai.env[KEY]` → `os.Getenv(KEY)` → `""`. Single-pass (no recursive
expansion).

**Removed in the 2026-04 kind split**: `${RECIPE_NAME}` (use
`${SCORE_NAME}`), `${MAX_ITERATION}` (loop bound is plateau-only).
Residual references resolve to empty string.

## Per-run nonces — `${HARNESS_NONCE_<NAME>}`

Recipes that probe end-state can be gamed: the AI sees the expected
key/value in the recipe and pre-sets it via any path (`podman exec`,
hardcoding, etc.) — bypassing the spirit of the test (e.g., "the SET
came FROM the client pod via cross-pod traffic").

The fix: per-run randomized nonces that the AI never sees. Recipes
reference `${HARNESS_NONCE_<NAME>}` tokens; the harness generates
fresh hex values via `crypto/rand` at run start, substitutes them
into a SECOND scenarios slice that flows ONLY into baseline
synthesis + per-iter scoring. The AI's prompt (rendered via
`${SCENARIOS}` and `${RECIPES}` from the un-substituted slice) shows
the placeholders verbatim.

```yaml
# Example: redis-client → redis cross-pod write with nonce
- name: cross-pod-set-from-client
  pod: redis-client
  steps:
    - then: "redis-client SETs bench:${HARNESS_NONCE_KEY} = ${HARNESS_NONCE_VALUE}"
      command: "redis-cli -h ov-redis SET bench:${HARNESS_NONCE_KEY} ${HARNESS_NONCE_VALUE}"
      stdout: { contains: [OK] }

- name: cross-pod-readback-on-server
  pod: redis
  steps:
    - then: "the random key landed in redis with the correct value"
      command: "redis-cli get bench:${HARNESS_NONCE_KEY}"
      stdout: { contains: ["${HARNESS_NONCE_VALUE}"] }
```

Per-run flow:
1. `HarnessRunLocalCmd.Run` calls `GenerateHarnessNonces` → walks
   merged scenarios, finds every unique `${HARNESS_NONCE_<NAME>}`,
   generates one 16-hex-char value per NAME via `crypto/rand` (64
   bits of entropy = brute-force-infeasible).
2. `SubstituteScenarioNonces` does a yaml round-trip with regex
   replace, producing the substituted slice.
3. The substituted slice goes into `synthesizeScoreBaseline` AND
   `RunRecipeScenariosLive` — so baseline IDs match runtime IDs
   (avoids false-Tampered cascade).
4. Prompt rendering (`${SCENARIOS}` / `${RECIPES}`) uses the
   un-substituted slice — the AI never sees the nonce values.

The same NAME used in multiple scenarios resolves to the SAME value
within a run (so SET in one pod and GET in another match). Different
runs get different values; even the same run on the same recipe is
unguessable.

Naming convention: `${HARNESS_NONCE_<UPPER_SNAKE>}` (e.g., `KEY`,
`VALUE`, `TOKEN1`, `RANDOM_PATH`). The harness regex is
`\$\{HARNESS_NONCE_([A-Z0-9_]+)\}`.

## Composition with `from:` — pull tests from existing entities

A recipe is a **requirement spec**: "to satisfy this deployment, all
of THESE tests must pass." Many tests already exist on the project's
layer / image / pod / vm entities (`tests:` and `deploy_tests:`
blocks). Rather than duplicating them inline as recipe scenarios, a
recipe can compose its requirement by pulling existing tests via the
`from:` block.

```yaml
recipe:
  prod-stack-ready:
    description: { feature: "App+redis stack is fully ready." }
    from:                                  # pull existing tests
      - kind: layer
        name: nodejs24
        pod:  app                          # harness container scenario.pod
        select: [node-installed, npm-installed]
      - kind: image
        name: arch-coder
        pod:  app                          # SAME pod as above — composition
        scope: [deploy]
        exclude: [legacy-port-9090]
      - kind: pod
        name: redis
        pod:  redis                        # different harness container
        prefix: redis                       # → "redis-<id>" scenario names
    scenario:                              # hand-written extras coexist
      - name: app-greets-redis
        pod: app
        depends_on: [redis-redis-running, npm-installed]   # cross-namespace deps
        steps: [...]
```

**Each `from:` entry expands at YAML-load time into N synthetic
`Scenario` records** — one per `Check` that survives the filter
pipeline. Each synthetic scenario has the supplied `pod:`, the
imported Check inline as its single `Step`, and a stable name
(`<prefix>-<check.id>` when the source has an id; otherwise
`<prefix>-<kind>-<origin>-<index>`).

### Fields

| Field | Required | Default | Purpose |
|---|---|---|---|
| `kind` | yes | — | One of `layer | image | pod | vm` |
| `name` | yes | — | Entity name (resolves in `cfg.Layers/Images/Pod/VM`) |
| `pod` | yes | — | Harness container name → becomes `scenario.pod` |
| `select` | no | (all) | Allow-list by `Check.ID` or synthesized name |
| `exclude` | no | (none) | Deny-list by `Check.ID` or synthesized name (applied AFTER select) |
| `scope` | no | `[layer, image, deploy]` | Which `LabelTestSet` sections to draw from |
| `prefix` | no | `""` | Scenario-name prefix to disambiguate collisions |
| `skip_live_only` | no | `true` | Drop `cdp/wl/dbus/vnc/mcp/record/spice/libvirt/k8s` verbs |

### Filter pipeline (per `from:` entry, in order)

1. **Section filter** via `scope:` — `layer` (layer-walked tests),
   `image` (entity-direct build-scope tests), `deploy` (entity-direct
   deploy-scope tests).
2. **Live-only verb filter** — drops checks whose verb requires live-
   container infrastructure (cdp/wl/etc.). Default: enabled.
3. **`select:` allow-list** — if non-empty, keep only checks whose ID
   (or synthesized name) matches.
4. **`exclude:` deny-list** — drop checks whose ID matches.

If the post-filter set is empty, the loader errors with a hint to
check filter ids and the `scope:` setting against the source entity.

### Per-kind expansion

| `kind` | Source | Mechanism |
|---|---|---|
| `layer` | Layer's own `tests:` slice | Direct read |
| `image` | `CollectTests(cfg, layers, name)` — full layer + base-image chain | Reuses image-build collector |
| `pod` | `cfg.Pod[name].Image`'s `CollectTests` chain + pod's own `tests:` and `deploy_tests:` | Pod wraps an image |
| `vm` | `cfg.VM[name]`'s `tests:` and `deploy_tests:` | VM's own + applies vm-target rule below |

### Scoring invariant — 1 `Check` = 1 ScenarioID = 1 point

**Imported tests are scored identically to hand-written tests.**
Every `Check` that survives the filter pipeline expands into exactly
one synthetic `Scenario` carrying exactly one `Step`. It contributes
ONE ScenarioID, identical weight to a hand-written scenario. A
`from:` block that imports 20 Checks adds 20 to the maximum
achievable score, identical to writing 20 inline scenarios. Plateau
detection, `${SCORE_DELTA}`, and solved-all detection see imported
and hand-written scenarios as indistinguishable — the harness scorer
cannot tell them apart by design.

This invariant is enforced by `TestImportedScenarioCountEqualsCheckCount`
in `ov/harness_recipe_from_test.go`. Bundling N imported Checks into
one fat `Scenario` with N inline `Step`s is FORBIDDEN — it would
collapse N points to 1 and kill the plateau gradient.

### Composition properties

- Multiple `from:` entries from DIFFERENT kinds compose freely.
- Multiple `from:` entries can target the SAME `pod:` (their imported
  scenarios all probe the same harness container) or DIFFERENT pods
  (multi-pod composed recipe — exactly the existing pattern from
  `tier4-redis`).
- Hand-written `scenario:` entries coexist and can `depends_on:`
  either imported or hand-written scenarios — same flat namespace.
- Imports appear FIRST in the merged `recipe.Scenario` slice;
  hand-written scenarios follow. This preserves authoring-order
  intuition (the `from:` block reads BEFORE `scenario:`).

### Validation rules

- `kind`, `name`, `pod` are all required per `from:` entry.
- `kind` ∈ `{layer, image, pod, vm}`.
- `name` MUST resolve in the matching catalog; error includes an
  available-names hint.
- `kind: vm` recipes can only be referenced by scores that target the
  same VM via `vm: <name>` — pod- or host-targeted scores can't reach
  VM-side tests through `podman exec`. Validator rejects mismatched
  scores at load time.
- Synthesized scenario names must not collide with each other or with
  hand-written scenarios in the same recipe; resolve via `prefix:` or
  rename the conflicting scenario.
- Empty result after `select`/`exclude` filtering is an error (likely
  a typo in the filter list).

### Worked examples in this repo

`harness.yml` ships two self-test recipes against the unified
`default` score:

- **`from-single-kind-selftest`** (phase 5) — one `from:` block per
  kind in isolation, each onto its own dedicated harness container.
  Failure here means per-kind expansion is broken.
- **`from-composition-selftest`** (phase 6) — two `from:` blocks
  onto the SAME container (composition), `select`+`exclude` filter
  pipeline, plus a hand-written scenario whose `depends_on:` crosses
  the imported/hand-written namespace boundary. Failure here means
  composition wiring or namespace resolution is broken.

Reference: `ov/harness_recipe_from.go` (expander),
`ov/harness_recipe_from_test.go` (per-kind, multi-kind, filter,
collision, scoring-invariant tests).

## Scenario dependencies — `depends_on:`

Some scenarios are only meaningful AFTER another scenario has passed.
Cross-pod tests are the canonical example: a `redis-cross-pod-readback-
on-server` (pod: `redis`) verifies that a `redis-cross-pod-set-from-
client` (pod: `redis-client`) actually wrote what the recipe claims.
Without ordering control, the bucket-by-pod scorer would happily run
the GET probe before the SET probe.

`depends_on:` is a list of scenario names that must pass first:

```yaml
- name: cross-pod-set-from-client
  pod: redis-client
  steps: [...]

- name: cross-pod-readback-on-server
  pod: redis
  depends_on: [cross-pod-set-from-client]   # named scenario must pass first
  steps: [...]
```

Scope: **intra-recipe only.** Each name must reference a scenario in
the SAME recipe. Cross-recipe references are rejected by
`validateHarnessSemantics` with a fuzzy "did you mean" hint. Cycles
are rejected with the cycle path (`A -> B -> A`).

Scoring honors deps via topological sort:
1. The scorer topo-sorts the merged scenarios with declaration-index
   tie-breaking (recipes read top-to-bottom by default; a `depends_on:`
   forces reordering only when needed).
2. After sort, the scorer groups consecutive same-pod scenarios into
   buckets — so a single `podman exec` per bucket. Cross-pod-dependent
   recipes get one extra `podman exec` per dependency boundary; tier4-
   redis with one `depends_on:` produces 3 buckets (redis A → redis-
   client B → redis C) instead of 2 (redis → redis-client). Trivial
   overhead.
3. Before each scenario runs, every entry in its `depends_on:` is
   checked. If any dep has verdict `fail` or `skipped`, the scenario
   is recorded with verdict `skipped` and `skipped_reason: "dep-unmet:
   <name>"`; its probes never execute. The skip cascades to dependents.

Skipped scenarios have their own bucket in `summary:` — they don't
count as solved or unsolved. The next iteration's `${SCENARIOS}`
listing surfaces the cascade so the AI knows which upstream to fix.

The same `${HARNESS_NONCE_*}` substitution applies to dependency-
ordered scenarios — a SET in pod A and a GET in pod B share the same
nonce values, generated once per run.

## Progressive recipe disclosure — `progressive: true`

Curriculum-style scoring: instead of dumping all four tier recipes on
the AI in iter1, the harness reveals one recipe per phase. Phase 1
gives the AI ONLY the easiest recipe. Once that's solved (or
plateaus), phase 2 brings phase 1's recipes BACK plus the next one.
This continues until every recipe has been added.

Enable via the score config:

```yaml
score:
  default:
    progressive: true                   # default: false (single-pass legacy mode)
    plateau_iteration: 3                # bound per phase, not per run
    recipes: [tier1-easy, tier2-medium, tier3-hard, tier4-redis]
```

Phase progression for the example score:

| Phase | In-scope recipes              | Scenarios | Hardness |
|-------|-------------------------------|-----------|----------|
| 1     | tier1-easy                    | 2         | trivial  |
| 2     | tier1-easy, tier2-medium      | 4         | easy     |
| 3     | tier1-easy, tier2-medium, tier3-hard | 6  | medium   |
| 4     | all 4                         | 11        | hard     |

Per-phase exit conditions:
- **solved-all**: every in-scope scenario passes → next phase begins.
- **plateau**: `plateau_iteration` consecutive iters of zero score
  delta → next phase begins regardless (carries partial state).

State persists across phases:
- Bench-pod is rebuilt FRESH at run start (existing host preflight),
  but phase boundaries do NOT touch it. Pods the AI deployed in phase
  1 stay running for phase 2's scoring → phase 2 reports `4/4` when
  tier1+tier2 are passing, not `2/2` for just the new tier2 surface.
- Nonces are run-scoped (generated once over the FULL recipe set), so
  a tier4 nonce is the same value across iters within phase 4.
- NOTES.md persists across phases (and across runs).

Result.yml under progressive scoring carries a `phases:` substructure:

```yaml
phases:
  - n: 1
    recipes: [tier1-easy]
    iterations_run: 1
    exit_reason: solved-all
    score: 2
    total: 2
  - n: 2
    recipes: [tier1-easy, tier2-medium]
    iterations_run: 1
    exit_reason: solved-all
    score: 4
    total: 4
  ...
phases_completed: 4
```

`iterations_run` at the top level becomes the SUM across all phases.
`best_score` reflects the LAST phase's totals. The non-progressive
shape (no `phases:` key) is unchanged for scores that don't opt in.

Per-phase iter dirs live under
`.harness/<score>/runs/<run-id>/phase<N>/iter<k>/` so each phase has
its own `iter1/`, `iter2/`, etc. without collision.

## Loop semantics

The loop runs as long as the AI keeps improving. Exit conditions:

| Exit reason | Condition |
|---|---|
| `solved-all` | All scenarios solved (k > 1) |
| `dry-run` | After iter 1 with `--dry-run` |
| `plateau` | `plateau_counter >= score.plateau_iteration` |
| `interrupted` | Context cancelled |

There is **no max-iteration ceiling**. Set `plateau_iteration` to bound
the run. With `plateau_iteration: 3`, a non-improving AI ends after 3
wasted iterations; a continually-improving AI runs forever (interrupt
with Ctrl-C).

### When source-code fixes can't apply mid-run

The harness orchestrator runs as a long-lived Go process (`ov harness
run-local`, PID 7 inside the bench-pod after preflight). Go binaries
are mmap'd at process start; **there is no in-process hot reload of
Go code**.

If the AI inside the bench-pod (or you, debugging an AI run)
identifies a bug in `ov/*.go`, applies a source fix, and rebuilds
`bin/ov`:

- The new binary lands on disk via the `/workspace` bind-mount, AND
- `go test ./...` against the new source compiles and runs the fixed
  code, BUT
- The RUNNING `ov harness run-local` PID still uses the OLD binary
  it mmap'd at process start. It cannot see the on-disk fix.

**Consequence:** The fix only takes effect on the NEXT
`ov harness run <score>` invocation, which forks a fresh PID that
mmap's the new binary. The current run's score will not improve as
a result of the fix.

**Recommended response (in the AI prompt):** Record the fix via
`bin/ov harness note append <score> "<short note about the fix>"`
and exit the iteration gracefully. Do NOT spin iterations 3+ trying
to make the running PID see the fix — plateau will trigger and
the run will exit anyway. The next run picks up the new binary
automatically.

The default-score prompt in `harness.yml` includes a paragraph
documenting this constraint to the AI explicitly so it knows to
exit gracefully rather than loop.

### Score-progress watchdog

The harness loop runs a per-iteration **score-progress watchdog**
that bounds each iteration by *scoring progress* (not wall clock).
Behavior:

- Every `progress_check_interval` (default **5m**), the harness
  probes the live deployments via the same `RunRecipeScenariosLive`
  code the end-of-iter scorer uses, and emits one host-side stderr
  line:
  `harness: progress [phase X/N iter Y] elapsed Nm — current score A/B (last improvement at HH:MM:SS)`.
- If the score has not increased in `progress_no_improvement_timeout`
  (default **30m**), the watchdog cancels the runner's context,
  emits
  `harness: watchdog [phase X iter Y] terminating AI runner — no scoring progress for ...`,
  and the iteration completes its end-of-iter scoring path normally.

**The watchdog is HIDDEN from the AI by construction** — it runs in
the harness Go process, never appears in the AI's prompt, never in
any tool the AI invokes. The `score.default.prompt:`'s "Take all
the time you need" promise is preserved from the AI's perspective;
the watchdog only fires when the AI is *genuinely* stuck (e.g. a
self-referential `pgrep -f` poll loop that never exits).

**Configuration** (per-AI; both optional, both default to the spec
values when empty):

```yaml
ai:
  claude:
    progress_check_interval: 5m
    progress_no_improvement_timeout: 30m
```

Set either to `0s` to disable that aspect (e.g. for fully-unbounded
development sessions where you want to debug an AI that's
legitimately taking hours). Set both to `0s` to disable the
watchdog entirely. The watchdog is independent of `ai.<name>.timeout:`
(absolute backstop) — both can be active simultaneously; whichever
fires first cancels the runner.

The watchdog only runs in **recipe-mode** (when `score.recipes:`
is non-empty). Image-test mode runs scoring after the runner exits,
so there's no live-score signal to poll mid-iter — those iterations
have no watchdog (rely on `ai.<name>.timeout:` if you need a cap).

Implementation: `ov/harness_watchdog.go` (`ProgressWatchdog` struct
+ `Run` goroutine), wired into `ov/harness_loop.go`'s runner
dispatch. Tests: `ov/harness_watchdog_test.go`.

## Multi-recipe sets + multi-pod scoring

A score's `recipes:` list IS a multi-recipe set. The harness:

1. Resolves each name against the recipe catalog.
2. Concatenates each recipe's scenarios in `recipes:` order.
3. **Buckets the merged scenarios by their `pod:` field.** Each unique
   pod gets its own `ContainerExecutor{ContainerName: "ov-" + pod}`;
   that bucket's scenarios run via `podman exec` against that
   container. Per-bucket sIdx is used for ScenarioIDs so baseline IDs
   match runtime IDs.
4. Renders the AI's prompt with `${RECIPES}` (per-recipe-grouped YAML
   block) and `${SCENARIOS}` (flat union). Both surface `pod:` per
   scenario so the AI knows which containers to spawn.

Validation:
- `recipes:` MUST be non-empty; duplicates rejected; each name MUST
  resolve.
- **Every scenario inside every recipe MUST set `pod:`.** Hard-error
  at load time otherwise.

Score = sum of solved scenarios across the whole set, regardless of
which recipe each scenario came from or which pod its steps targeted.

### Three pod-topology patterns supported

| Pattern | Recipes | Pods AI spawns |
|---|---|---|
| Single-pod-per-recipe | `recipe.X` with all scenarios `pod: pod-X` | one pod (`pod-X`) |
| Pod-shared-across-recipes | `recipe.A` and `recipe.B` both have scenarios `pod: shared-pod` | one pod (`shared-pod`) — same container, more scenarios scored against it |
| Multi-pod-per-recipe | `recipe.M` with scenarios `pod: pod-A` AND scenarios `pod: pod-B` | two pods, both probed; one recipe spans both |

The harness handles all three uniformly via the bucket-by-pod scoring
loop. The AI reads `${RECIPES}`, collects the unique set of `pod:`
values, and `ov deploy add`s each.

## Setup — one-time per-score pod

```bash
ov deploy add bench-pod fedora-coder --bind project=$PWD
ov start bench-pod
# Edit harness.yml: set score.<name>.pod: bench-pod
ov harness sync-credential <name>
ov harness run <name> --ai claude
```

For host targets:

```bash
# Edit harness.yml:
#   score.<name>.host: true
#   score.<name>.disposable: true
ov harness run <name> --ai claude
```

For VM targets:

```bash
# score.<name>.vm: my-vm-name (a kind:vm entry)
ov vm start my-vm-name
ov harness sync-credential <name>           # SCP creds into the VM
ov harness run <name> --ai claude
```

## When to Use This Skill

**MUST be invoked** before authoring `harness.yml`, debugging the
iteration loop, interpreting result files, or modifying any
`ov/harness_*.go` / `ov/ai_config.go` / `ov/migrate_harness.go` file.

## Cross-references

- `/ov:test` — `ov test` and `ov image test` semantics; the verb catalog the harness scorer consumes
- `/ov:layer`, `/ov:image` — the YAML surfaces the AI edits
- `/ov:migrate` — `ov migrate harness` command surface
- `/ov-layers:ov-mcp` — the `/workspace` bind-mount convention used by pod targets
- `/ov-layers:container-nesting` — the rootless-nested-podman recipe that lets pod targets build images
- `/ov-images:fedora-coder`, `/ov-images:arch-coder` — canonical bench-pod images
- `/ov-dev:go` — Go-side implementation map (`harness_*.go`, `ai_config.go`, `migrate_harness.go`)
- `/ov-dev:cutover-policy` — hard-cutover policy that governed both this and the prior dispatcher-removal cutover
- `/ov-dev:disposable` — disposability rule enforced for `host: true` scores
