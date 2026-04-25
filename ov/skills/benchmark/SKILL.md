---
name: benchmark
description: |
  MUST be invoked before any work involving: `ov benchmark` commands, the
  `benchmark:` section in overthink.yml, the iteration-until-plateau
  scoring loop, AI-runner credential sync, the `ovbench/<run-id>` git
  branch convention, or the seven benchmark verdicts (solved / partial /
  unchanged / regressed / tampered / retagged / added).
  Covers runner configuration, plateau semantics, deploy-kind dispatch
  (host/pod/vm; k8s rejected in v1), worktree lifecycle, credential sync
  with no-worktree-leak guarantee, `ov image test --format yaml` scoring,
  fingerprint-based tampering detection, and the AI-facing helpers
  (`ov benchmark scope`, `ov benchmark last-test-tag`,
  `ov benchmark self-evaluate`).
---

# benchmark — Score AI agents against BDD scenarios via iterative rebuild-test loops

## Overview

`ov benchmark` runs an AI agent (claude / codex / gemini / any configured runner) **inside** an existing host/pod/vm deployment, lets it edit source in a dedicated git worktree to solve pending BDD scenarios, and iterates until the solved-count plateaus for N consecutive iterations. The deployment IS the sandbox. The worktree IS the workspace. `ov image test --format yaml` IS the scorer.

The feature composes two prior cutovers:

1. **BDD description cutover** (commit `64fd7d7`) — every `kind:` entity (layer / image / pod / vm / k8s / host / deployment) carries a structured `description:` block with Gherkin-shaped `scenarios:`. Each scenario step embeds an existing `ov test` Check, so scenarios are runnable verdicts, not prose.
2. **Unified deploy targets (schema v4)** — `ov deploy add <name> <ref>` produces one of four sandbox flavours (host / pod / vm / k8s) with a shared shell surface (`ov cmd` / `ov shell` / `ov ssh` / `ov vm ssh`).

`ov benchmark` fires the AI inside one of those sandboxes, rebuilds the image from the worktree after every iteration, runs `ov image test <ovbench-tag> --format yaml` to score, and stops when the AI stops improving.

## Quick Reference

| Command | Purpose |
|---|---|
| `ov benchmark run <deployment>` | Main driver — iterate until plateau |
| `ov benchmark list` | Past runs under `.benchmark/` |
| `ov benchmark list-runners` | Configured runners from `overthink.yml` `benchmark:` section |
| `ov benchmark report [<run-id>]` | Re-render a past `report.yml` (default: latest) |
| `ov benchmark scope` | **AI-facing**: current iteration's scope YAML |
| `ov benchmark last-test-tag` | **AI-facing**: prior iteration's image tag (for cheap self-inspection) |
| `ov benchmark self-evaluate` | **AI-facing, expensive**: rebuilds current worktree + runs `ov image test` |

## The four design pillars

### 1. Everything through `ov` + YAML. No JSON.

Runner config lives in `overthink.yml` under a new `benchmark:` key. The prompt template is a YAML multiline string on the same block. The per-run report persists as `.benchmark/<run-id>/report.yml`. Per-iteration state persists as `iter<k>/score.yml` + `iter<k>/test-output.yaml`. Fingerprints are sha256 hex strings inside those YAML files. There is zero JSON on the benchmark surface.

### 2. Git worktree per run — the workspace concept

Every benchmark run creates:

```
git worktree add .benchmark/<run-id>/worktree HEAD -b ovbench/<run-id>
```

The worktree is the AI's workspace. The deployment's `/workspace` bind-mount (from the `ov-mcp` layer) makes the worktree reachable inside pod/vm deploys at `/workspace/.benchmark/<run-id>/worktree/`. Host deploys reach the same dir at its absolute host path.

After each iteration the harness stages + commits with **no `--no-verify`** (hooks run):

```
git commit -m "iter<k>: score=N, solved=[id1,id2,...]" --allow-empty
```

Post-benchmark, `git log ovbench/<run-id>` is the audit trail. `git diff main..ovbench/<run-id>` is the total AI delta. Cleanup is `git worktree remove` + `git branch -D`.

Damage isolation: if the AI breaks `layer.yml` parsing or any other invariant, the damage lives on the `ovbench/<run-id>` branch. `main` stays clean.

### 3. `ov image test --format yaml` is the scorer

After each iteration's rebuild into `ovbench/<run-id>-iter<k>:<image>`, the harness shells out:

```
ov image test ovbench/<run-id>-iter<k>:<image> --format yaml
```

parses the output via `ParseOvTestOutput` in `benchmark_score.go`, and counts scenarios with verdict `solved` as the iteration score. Improvements to `ov test` flow through the benchmark automatically.

### 4. Iteration loop with plateau-driven termination

For each iteration `k` (1-indexed):

1. Refresh scope YAML (still-unsolved scenarios + score-trajectory history).
2. Render prompt with iteration-specific tokens substituted.
3. Dispatch runner into deploy (via `ov cmd` / `ov ssh` / `ov vm ssh`).
4. Rebuild image from worktree into `ovbench/<run-id>-iter<k>:<image>`.
5. Shell out to `ov image test <iter-tag> --format yaml`; parse.
6. Classify each pre-AI scenario (7-way verdict). Compute `score_k = count(solved)`.
7. Commit iteration on the `ovbench/<run-id>` branch.
8. Plateau check: `score_k > best_score` → reset counter; else increment.
9. Exit when plateau counter ≥ `--plateau-iterations` (default 3).

No overall timeout. Per-runner `timeout:` bounds individual AI invocations (default 30m).

## Config shape — `benchmark:` in `overthink.yml`

```yaml
benchmark:
  runners:
    - name: claude
      command: [claude, -p, "${PROMPT}"]
      prompt_via: argv               # argv (default) | stdin | file
      env:
        ANTHROPIC_API_KEY: "${ANTHROPIC_API_KEY}"
      timeout: 20m                   # default 30m when omitted
      credentials:
        - src: ~/.claude/.credentials.json
          dst: ~/.claude/.credentials.json
    - name: codex
      command: [codex, exec, "${PROMPT}"]
      credentials:
        - {src: ~/.config/codex/auth.json, dst: ~/.config/codex/auth.json, optional: true}
    - name: gemini
      command: [gemini, chat, "${PROMPT}"]
      credentials:
        - {src: ~/.config/gcloud/application_default_credentials.json, dst: ~/.config/gcloud/application_default_credentials.json, optional: true}

  prompt: |
    You are benchmarking BDD-scenario-solving inside an overthink project.
    This is iteration ${ITERATION} of up to ${MAX_ITERATIONS}. Your current
    best score is ${BEST_SCORE}; plateau counter is ${PLATEAU_COUNTER} of
    ${PLATEAU_ITERATIONS}.

    Your cwd is the git worktree at ${WORKSPACE}. Edits you make are
    captured per-iteration as git commits on branch ovbench/${RUN_ID}.

    MCP tools: ${MCP_ENDPOINT}

    Read scope:              ov benchmark scope
    Inspect scenario:        ov feature list <origin>
    Validate edits:          ov feature validate <origin>
    See prior test output:   ov image test $(ov benchmark last-test-tag) --format yaml
    Exit when done with this iteration.
```

### Substitution tokens

`${PROMPT}`, `${PROMPT_FILE}`, `${WORKSPACE}`, `${TARGET_IMAGE}`, `${TARGET_DEPLOYMENT}`, `${RUN_ID}`, `${ITERATION}`, `${MAX_ITERATIONS}`, `${PLATEAU_ITERATIONS}`, `${PLATEAU_COUNTER}`, `${BEST_SCORE}`, `${MCP_ENDPOINT}`, `${TAGS}`, `${DEADLINE}`, `${TIMEOUT}`, plus any `${X}` that falls through to `os.Getenv("X")`. Unresolved tokens expand to the empty string.

### Default skeleton (what a fresh project ships)

Copy verbatim:

```yaml
benchmark:
  runners:
    - name: claude
      command: [claude, -p, "${PROMPT}"]
      credentials:
        - {src: ~/.claude/.credentials.json, dst: ~/.claude/.credentials.json}
    - name: codex
      command: [codex, exec, "${PROMPT}"]
      credentials:
        - {src: ~/.config/codex/auth.json, dst: ~/.config/codex/auth.json, optional: true}
    - name: gemini
      command: [gemini, chat, "${PROMPT}"]
      credentials:
        - {src: ~/.config/gcloud/application_default_credentials.json, dst: ~/.config/gcloud/application_default_credentials.json, optional: true}
```

Each runner inherits `timeout: 30m` from the Go-level default. Credentials sync the MINIMUM file the AI CLI needs to authenticate non-interactively — typically a single refresh-token file. Do not sync whole config directories: `~/.claude` alone is ~430 MB / 7k files, and only `.credentials.json` is load-bearing for `claude -p` auth.

## Supported deploy kinds

| Target | Exec verb | Workspace (inside deploy) | Rebuild path |
|---|---|---|---|
| `pod` | `ov cmd <deploy> --` | `/workspace/.benchmark/<run-id>/worktree` | Host runs `ov -C <host-worktree> image build` |
| `host` | `ov ssh <alias>` (if SSH-addressable) or local shell | Host absolute worktree path | Host runs `ov image build` |
| `vm` | `ov vm ssh <vm> --` | `/workspace/.benchmark/<run-id>/worktree` via virtiofs/9p | Host runs `ov image build` |
| `k8s` | — | — | **REJECTED in v1 with `ErrK8sUnsupported`.** Nested-build complexity; deferred. |

**Preflight requirements** on the target deployment:

- `ov-mcp` layer present (unless `--no-mcp` is passed).
- AI-CLI layer matching the runner's `command[0]` (preflight probes `command -v <bin>` inside the deploy).
- Writable `/workspace` bind-mount mapping to the project root.

## The seven verdicts

Every pre-AI scenario is classified after each iteration:

| Verdict | Trigger | Counts toward score? |
|---|---|---|
| `solved` | Baseline fail/error, post pass, fingerprint unchanged, no pending steps | **Yes** — the sole score contributor |
| `partial` | Pending-step count decreased, or some steps green but scenario still fails | No |
| `unchanged` | No delta between baseline and post | No |
| `regressed` | Baseline passed; post fails | No |
| `tampered` | Fingerprint changed AND post pass (AI weakened the scenario) | **No** — flagged loudly |
| `retagged` | Body unchanged, only tags differ | No (soft warning) |
| `added` | Post-only scenario (AI added a new one) | No (reported separately) |

## Credential sync — preflight-once, never per-iteration

AI CLIs typically hold auth state in the user's `$HOME` (`~/.claude/` for Claude Code, `~/.config/gcloud/` for Gemini ADC, `~/.config/codex/` for Codex). The benchmark syncs these into the deployment **once** at preflight via `Dispatcher.SyncCredentials`. Not per iteration — re-syncing mid-run would clobber OAuth refresh tokens the AI rotated during its work.

Copy semantics per deploy kind:

| Kind | Mechanism |
|---|---|
| `host` (local) | `cp -a` (local filesystem; handled by `syncCredentialsLocal`) |
| `host` (SSH) | `rsync -a …` over ssh |
| `pod` | `podman cp <src> <container>:<dst>` |
| `vm` | `rsync -a -e 'ov vm ssh <name> --' …` |

**Privacy invariant (enforced by `TestSyncCredentials_EndToEnd`):**

1. Credentials land at the deploy's `$HOME` (or resolved `dst`).
2. Credentials NEVER land in the worktree.
3. `git status` on the worktree stays clean post-sync.

The dispatcher's `SyncCredentials` contract explicitly forbids any code path that would copy credential material into the worktree. Violating that invariant would leak tokens into the `ovbench/<run-id>` commit history.

## AI-facing feedback loop

Each iteration the AI sees a trajectory-aware scope YAML via `ov benchmark scope`:

```yaml
run_id: 20260425-101500-abc123
iteration: 3
max_iterations: 50
plateau_iterations: 3
plateau_counter: 1
best_score: 2
target_image: fedora-ov
target_deployment: hermes-disposable
history:
  - k: 1
    score: 1
    solved_ids: [desc:layer:sshd:0]
    runtime: 12m14s
  - k: 2
    score: 2
    solved_ids: [desc:layer:sshd:0, desc:layer:ssh-client:0]
    plateau_counter_after: 0
scenarios:                           # still-unsolved only
  - id: desc:layer:foo:0
    origin: layer:foo
    baseline_verdict: fail
    pending_steps_current: 2
```

The AI can also run `ov image test $(ov benchmark last-test-tag) --format yaml` to re-inspect the prior iteration's detailed results WITHOUT triggering a rebuild (the previous iteration's `ovbench/<run-id>-iter<k-1>:<image>` tag is still in local storage).

Opt-in expensive self-check: `ov benchmark self-evaluate` rebuilds the current worktree into a throwaway tag and runs `ov image test` against it, printing the score. It is **pure** — does NOT mutate the active iteration's scope or committed history. The authoritative score always comes from the harness at end-of-iteration.

## Filesystem layout under `.benchmark/<run-id>/`

```
.benchmark/<run-id>/
├── worktree/                    # git worktree on branch ovbench/<run-id>
│   └── .benchmark/
│       ├── scope.yml            # mirror of active iteration's scope
│       └── prompt.md            # mirror of active iteration's prompt
├── iter1/
│   ├── scope.yml
│   ├── prompt.md
│   ├── prompt-arg.md            # when prompt_via: file
│   ├── runner.log
│   ├── build.log
│   ├── test-output.yaml         # raw `ov image test --format yaml` output
│   └── score.yml                # classified verdicts + fingerprints
├── iter2/
│   └── ...
└── report.yml                   # aggregated final report
```

Nothing outside `.benchmark/<run-id>/` and the `ovbench/<run-id>` git branch is written by the harness.

## Runner argv + env injection

Every AI invocation runs with these env vars automatically injected (merged with runner `env:`):

| Var | Value |
|---|---|
| `OV_BENCHMARK_RUN_ID` | Run identifier |
| `OV_BENCHMARK_ITERATION` | Current iteration number |

These are what `ov benchmark scope` and `ov benchmark last-test-tag` read to locate the active run.

## Exit conditions

| Reason | Trigger |
|---|---|
| `plateau` | `plateau_counter >= --plateau-iterations` |
| `ceiling` | `k == --max-iterations` (when non-zero) |
| `solved-all` | Every pre-AI scenario reached `solved` |
| `dry-run` | `--dry-run` exited after iter1 scope+prompt rendering |
| `interrupted` | SIGINT or context cancellation |

## When to Use This Skill

**MUST be invoked** before authoring runner configs, debugging benchmark loops, interpreting reports, or modifying any `ov/benchmark_*.go` file. Invoke BEFORE reading the source code.

## Cross-references

- `/ov:test` — `ov test` and `ov image test` semantics; the verb catalog the scorer consumes
- `/ov:layer`, `/ov:image` — the YAML surfaces the AI edits
- `/ov-layers:ov-mcp` — the `/workspace` bind-mount convention benchmark relies on
- `/ov-dev:go` — Go-side implementation map (including `benchmark_score.go`, `benchmark_config.go`, `benchmark_worktree.go`, `benchmark_dispatch.go`, `benchmark_loop.go`, `benchmark_cmd.go`)
- `/ov-dev:cutover-policy` — the hard-cutover rules that govern how the benchmark cutover was shipped
- `/ov-dev:disposable` — `disposable: true` authorization rules (relevant because benchmarks target disposable deploys)
