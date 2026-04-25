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

This is the **2026-04-26 successor** to `ov benchmark`. The old `benchmark:`
block AND the post-rename "fat recipe" (recipe carrying runner fields) are
no longer accepted; `ov migrate harness` produces a `harness.yml` include
carrying THREE kind-entities side by side:

- `kind: ai`     — reusable AI-CLI catalog (unchanged).
- `kind: recipe` — **pure spec**: description + BDD scenarios. No
  targets, no AI, no prompt, no plateau.
- `kind: score`  — **runner config**: target (`pod`/`vm`/`host`), AI
  list, plateau, prompt, deployment, MCP endpoint, notes, env, plus
  `recipes:` — the ordered list of recipes it evaluates against in
  every iteration.

A score with `recipes: [a]` is a single-recipe run. A score with
`recipes: [a, b, c]` is a multi-recipe set: every iteration the AI is
shown all three recipes via `${RECIPES}` and the harness scores against
the union (sum of solved scenarios across a, b, c).

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
    description: { feature: "Tier 1 — easy: marker file." }
    scenario:
      - name: tier1-marker-file-exists
        steps:
          - then: "/etc/ov-bench-marker exists"
            file: /etc/ov-bench-marker
            exists: true

  tier2-medium:
    description: { feature: "Tier 2 — medium: HTTP service." }
    scenario: [...]

  tier3-hard:
    description: { feature: "Tier 3 — hard: package + state." }
    scenario: [...]

score:                                  # kind: score — runner config
  default:
    description: { feature: "Default benchmark — three difficulty tiers." }
    host: true
    disposable: true                    # required for host: true
    ai: [claude]
    plateau_iteration: 3                # the only loop bound
    target_image: "bench-target"
    deployment: "harness-default"       # AI must `ov deploy add` this
    notes: true
    mcp_endpoint: "http://localhost:18765/mcp"
    env: {}
    recipes: [tier1-easy, tier2-medium, tier3-hard]   # ordered set
    prompt: |
      ...
```

## Recipe vs Score — what goes where

| Concern | Lives on `recipe:` | Lives on `score:` |
|---|---|---|
| Scenarios (BDD) | YES | NO |
| `description:` | YES (the spec's narrative) | YES (the run's narrative) |
| Target — `host:`/`pod:`/`vm:` | NO | YES (xor: exactly one) |
| `disposable:` | NO | YES (required when `host: true`) |
| `ai:` list | NO | YES |
| `plateau_iteration:` | NO | YES |
| `tag:` (Gherkin filter) | NO | YES |
| `target_image:`, `deployment:` | NO | YES |
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
| `${SCENARIOS}` | Flat union of all referenced recipes' scenarios |
| `${RECIPES}` | Per-recipe-grouped YAML block (one entry per `score.recipes`) |
| `${DEPLOYMENT}` | Score's `deployment:` |

**Substitution precedence**: well-known token → `score.env[KEY]` →
`ai.env[KEY]` → `os.Getenv(KEY)` → `""`. Single-pass (no recursive
expansion).

**Removed in the 2026-04 kind split**: `${RECIPE_NAME}` (use
`${SCORE_NAME}`), `${MAX_ITERATION}` (loop bound is plateau-only).
Residual references resolve to empty string.

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

## Multi-recipe sets via `score.recipes`

A score's `recipes:` list IS a multi-recipe set. The harness:

1. Resolves each name against the recipe catalog.
2. Concatenates each recipe's scenarios in `recipes:` order, stamping
   each scenario with its source recipe (`Scenario.SourceRecipe`).
3. Scores against the merged set every iteration.
4. Renders the AI's prompt with `${RECIPES}` (per-recipe-grouped YAML
   block) and `${SCENARIOS}` (flat union).

Validation:
- `recipes:` MUST be non-empty.
- Each name in `recipes:` MUST resolve to a defined recipe in the same
  `harness.yml`.
- Duplicates in `recipes:` are rejected.

Score = sum of solved scenarios across the whole set; increases
monotonically as the AI solves more, regardless of which recipe each
scenario came from.

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
