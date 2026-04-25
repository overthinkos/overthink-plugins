# harness — Drive AI agents through BDD-scenario iteration cycles

## Overview

`ov harness` is the project's iterative AI driver. Same loop, two modes:

- **Benchmarking** — score an AI against pending BDD scenarios; iterate until plateau.
- **BDD development** — write failing scenarios first, let the AI drive code that makes
  them pass.

Both modes share the machinery: per-iteration git clone + AI-CLI invocation + nested
`ov image build` + `ov image test --format yaml` scoring + plateau-bounded loop. Only
the prompt + termination policy differ.

This is the **2026-04 successor** to `ov benchmark`. The old `benchmark:` block in
`overthink.yml` is no longer accepted; `ov migrate harness` produces a `harness.yml`
include carrying two new kind-entities side by side: `kind: ai` (the reusable AI-CLI
catalog) and `kind: recipe` (named harness recipes).

## Quick Reference

| Command | Purpose |
|---|---|
| `ov harness run <recipe>` | Drive an AI through iteration cycles for the named recipe |
| `ov harness sync-cred <recipe>` | Copy AI credentials into the recipe's target |
| `ov harness list-ai` | Show configured AIs (with version_command + credentials count) |
| `ov harness list-recipe` | Show configured recipes (with target + AI list + plateau) |
| `ov harness list` | List past runs under `.harness/<recipe>/` |
| `ov harness report [<recipe>] [<calver>]` | Print a past `result.<calver>.yml` |
| `ov harness scope` | **AI-facing**: print current iteration scope |
| `ov harness last-tag` | **AI-facing**: prior iteration's image tag |
| `ov harness note read [<recipe>]` | Print persistent NOTES.md memory |
| `ov harness note append <recipe> <text>` | Atomic timestamped note append |
| `ov migrate harness` | One-shot legacy `benchmark:` → `harness.yml` migrator |

## `harness.yml` shape — `ai:` catalog + `recipe:` recipes

```yaml
ai:                                            # kind: ai — reusable AI-CLI catalog
  claude:
    description:
      feature: "Anthropic Claude Code CLI"
    command: [claude, -p, --dangerously-skip-permissions, "${PROMPT}"]
    prompt_via: argv                           # argv | stdin | file
    version_command: [claude, --version]       # captured once per run
    timeout: 30m
    credential:
      - {src: ~/.claude/.credentials.json, dst: ~/.claude/.credentials.json}

recipe:                                        # kind: recipe — named harness configurations
  bench-claude:
    description:
      feature: "Benchmark recipe — score claude until plateau"
    pod: bench-pod                             # exactly ONE of {pod, vm, host}
    ai: [claude]                               # eligible AI list
    plateau_iteration: 3
    max_iteration: 50
    tag: ""                                    # Gherkin filter expression
    target_image: ""                           # "" = pod's own image
    notes: true                                # persistent NOTES.md across runs
    mcp_endpoint: "http://localhost:18765/mcp" # ${MCP_ENDPOINT}; "" disables
    env:                                       # arbitrary tokens; KEY → ${KEY}
      JUPYTER_MCP: "http://localhost:8888/mcp"
    prompt: |
      ...

  bdd-dev-on-host:
    description:
      feature: "Development recipe — make failing scenarios pass on the host"
    host: true
    disposable: true                           # required for host: true (per /ov-dev:disposable)
    ai: [claude]
    plateau_iteration: 1
    max_iteration: 20
    prompt: |
      ...
```

## Where the harness runs — `pod:` / `vm:` / `host:` (exactly one)

| Field | Value | Meaning | Other `ov` commands |
|---|---|---|---|
| `pod: <name>` | name of a running pod deployment | Run iterations inside this pod via `podman exec` | `ov start <name>` / `ov status <name>` |
| `vm: <name>` | name of a VM | Run iterations inside this VM via SSH | `ov vm start <name>` / `ov vm ssh <name>` |
| `host: true` | (boolean) | Run iterations on the local machine directly | (none — workspace is `$PWD`) |

Validation: `ov image validate` (and load-time checks) hard-error if a recipe has zero
or more-than-one of these fields set. CLI overrides via mutually-exclusive `--on-pod
NAME` / `--on-vm NAME` / `--on-host` (xor group).

`host: true` requires `disposable: true` on the recipe — host iterations destructively
edit `$PWD`'s git tree on a per-run branch (same risk as `ov rebuild`). Explicit-only.

## Filesystem layout under `.harness/`

```
.harness/
└── <recipe-name>/
    ├── note/
    │   └── NOTES.md                 # persistent memory across runs (notes: true)
    ├── results/
    │   ├── result.2026.115.1430.yml # one per ov harness run, calver = YYYY.DDD.HHMM (UTC)
    │   └── result.2026.115.1530.yml
    └── runs/
        └── <run-id>/                # ephemeral build artifacts
            ├── repo/                # per-run git clone, branch ovharness/<run-id>
            ├── iter1/
            │   ├── runner.log
            │   ├── build.log
            │   ├── prompt.md
            │   ├── prompt-arg.md
            │   ├── scope.yml
            │   ├── test-output.yaml
            │   └── score.yml
            └── iter2/
```

`.harness/` is in `.gitignore`. The `result.<calver>.yml` file is the load-bearing
durable artifact; the `runs/<run-id>/` tree is best-effort persistence.

### Result file body

```yaml
schema: 1
recipe: bench-claude
calver: 2026.115.1430
run_id: 20260425-143000-abc123
ai: claude
ai_version:
  claude: "0.5.2 (Claude Code, Sonnet 4.6)"
where:
  kind: pod                       # pod | vm | host
  name: bench-pod                 # absent when kind: host
target_image: fedora-coder
plateau_iteration: 3
max_iteration: 50
mcp_endpoint: "http://localhost:18765/mcp"
exit_reason: plateau              # plateau | ceiling | solved-all | dry-run | interrupted
final_score: 4
best_score: 4
ovharness_branch: ovharness/20260425-143000-abc123
iteration:
  - k: 1
    score: 1
    solved_id: [desc:layer:sshd:0]
    runtime: 12m14s
    plateau_counter_after: 0
  - k: 2
    ...
```

## Memory — persistent notes across runs

Each recipe gets `.harness/<recipe>/note/NOTES.md`, shared across runs. The harness
exposes:

| Verb | Behavior |
|---|---|
| `ov harness note read [<recipe>]` | Print the file. Defaults to `$OV_HARNESS_RECIPE` env (set during iterations). |
| `ov harness note append <recipe> <text>` | `O_APPEND` write with header `## <calver> run=<run-id> iter=<k> ai=<name>`. |

The default migrated prompt template prepends a `== Memory ==` block that expands
`${NOTES}` (the file's content, snapshotted per iteration) and tells the AI it can
write notes via `ov harness note append`.

Disable per-recipe with `notes: false` — `${NOTES}` then expands to empty and `ov
harness note append/read` returns "notes disabled for recipe X".

## Substitution tokens

| Well-known token | Source |
|---|---|
| `${PROMPT}`, `${PROMPT_FILE}` | Rendered prompt text / file path |
| `${WORKSPACE}` | Per-run repo dir |
| `${RUN_ID}`, `${RECIPE_NAME}`, `${AI_NAME}` | Run identity |
| `${TARGET_KIND}`, `${TARGET_NAME}` | "pod" / "vm" / "host" + name |
| `${TARGET_IMAGE}`, `${TAG}` | Recipe inputs |
| `${ITERATION}`, `${MAX_ITERATION}`, `${PLATEAU_ITERATION}`, `${PLATEAU_COUNTER}`, `${BEST_SCORE}` | Iteration state |
| `${MCP_ENDPOINT}` | Recipe's `mcp_endpoint:` (default `http://localhost:18765/mcp`) |
| `${NOTES}` | Per-iteration NOTES.md snapshot |
| `${TIMEOUT}`, `${DEADLINE}` | Resolved per-AI timing |

**Substitution precedence**: well-known token → `recipe.env[KEY]` → `ai.env[KEY]` →
`os.Getenv(KEY)` → `""`. Single-pass (no recursive expansion).

## Setup — one-time per-recipe pod

```bash
ov deploy add bench-pod fedora-coder --bind project=$PWD
ov start bench-pod
# Edit harness.yml: set recipe.bench-claude.pod: bench-pod
ov harness sync-cred bench-claude
ov harness run bench-claude --ai claude --max-iteration 5
```

For host targets:

```bash
# Edit harness.yml:
#   recipe.bdd-dev-on-host.host: true
#   recipe.bdd-dev-on-host.disposable: true
ov harness run bdd-dev-on-host --ai claude --max-iteration 1
```

For VM targets:

```bash
# Recipe declares: recipe.bench-vm.vm: my-vm-name (a kind:vm entry)
ov vm start my-vm-name
ov harness sync-cred bench-vm                  # SCP creds into the VM via SSH
ov harness run bench-vm --ai claude
```

## When to Use This Skill

**MUST be invoked** before authoring `harness.yml`, debugging the iteration loop,
interpreting result files, or modifying any `ov/harness_*.go` / `ov/ai_config.go` /
`ov/migrate_harness.go` file.

## Cross-references

- `/ov:test` — `ov test` and `ov image test` semantics; the verb catalog the harness scorer consumes
- `/ov:layer`, `/ov:image` — the YAML surfaces the AI edits
- `/ov:migrate` — `ov migrate harness` command surface
- `/ov-layers:ov-mcp` — the `/workspace` bind-mount convention used by pod targets
- `/ov-layers:container-nesting` — the rootless-nested-podman recipe that lets pod targets build images
- `/ov-images:fedora-coder`, `/ov-images:arch-coder` — canonical bench-pod images
- `/ov-dev:go` — Go-side implementation map (`harness_*.go`, `ai_config.go`, `migrate_harness.go`)
- `/ov-dev:cutover-policy` — hard-cutover policy that governed both this and the prior dispatcher-removal cutover
- `/ov-dev:disposable` — disposability rule enforced for `host: true` recipes
