---
name: benchmark
description: |
  MUST be invoked before any work involving: `ov benchmark` commands, the
  `benchmark:` section in overthink.yml, the iteration-until-plateau
  scoring loop, AI-runner credential sync into the bench-pod, the
  `ovbench/<run-id>` git branch convention, or the seven benchmark
  verdicts (solved / partial / unchanged / regressed / tampered /
  retagged / added).
  Covers runner configuration, plateau semantics, the thin-host /
  fat-pod architecture, the per-run clone model (no host worktree),
  one-shot credential sync, `ov image test --format yaml` scoring run
  inside the pod's nested podman, fingerprint-based tampering detection,
  and the AI-facing helpers (`ov benchmark scope`,
  `ov benchmark last-test-tag`, `ov benchmark self-evaluate`).
---

# benchmark — Score AI agents against BDD scenarios via iterative rebuild-test loops

## Overview

`ov benchmark` runs an AI agent (claude / codex / gemini / any configured runner) **inside a persistent pod** that clones the project's repo and iterates against the pending BDD scenarios until the solved-count plateaus. The pod IS the sandbox. The pod's per-run git clone IS the workspace. `ov image test --format yaml` (run inside the pod against a freshly built iteration tag) IS the scorer.

**Architecture: thin-host / fat-pod.** The host's `ov benchmark run <pod>` is a pure forwarder — it validates the pod, generates a run-id, and execs `ov benchmark run-local` inside the pod via `podman exec`. Every interesting piece (clone, build, test, score, classify, fingerprint, commit) happens *exclusively inside the pod* using the pod's own `ov` binary. After completion the host `podman cp`s the run dir back so `ov benchmark list` and `ov benchmark report` work host-side.

The feature composes three prior cutovers:

1. **BDD description cutover** — every `kind:` entity carries `description.scenarios:` Gherkin blocks, each step embedding an existing `ov test` Check.
2. **Unified deploy targets (schema v4)** — `ov deploy add <name> <ref>` produces a long-lived pod with the bind-mounted `/workspace`.
3. **Container-nesting + ov-mcp + AI-CLI layers** — `fedora-coder` and `arch-coder` already ship rootless nested podman, the MCP gateway, and claude/codex/gemini, so a benchmark pod is a one-line `ov deploy add` away.

## Quick Reference

| Command | Purpose |
|---|---|
| `ov benchmark run <pod>` | Host forwarder — dispatches into the pod and streams output |
| `ov benchmark run-local <image> --run-id <id>` | Pod-side iteration driver (HIDDEN; the host invokes this) |
| `ov benchmark sync-credentials <pod>` | One-shot copy of AI-CLI credentials into the bench-pod |
| `ov benchmark list` | Past runs under `.benchmark/` (host-mirrored) |
| `ov benchmark list-runners` | Configured runners from `overthink.yml` `benchmark:` |
| `ov benchmark report [<run-id>]` | Re-render a past `report.yml` (default: latest) |
| `ov benchmark scope` | **AI-facing**: current iteration's scope YAML |
| `ov benchmark last-test-tag` | **AI-facing**: prior iteration's image tag |
| `ov benchmark self-evaluate` | **AI-facing, expensive**: rebuilds current clone + runs `ov image test` |

## The four design pillars

### 1. Everything through `ov` + YAML. No JSON.

Runner config lives in `overthink.yml` under `benchmark:`. The prompt template is a YAML multiline string on the same block. The per-run report persists as `.benchmark/<run-id>/report.yml`. Per-iteration state persists as `iter<k>/score.yml` + `iter<k>/test-output.yaml`. Fingerprints are sha256 hex strings inside those YAML files. Zero JSON on the benchmark surface.

### 2. Per-run git clone inside the pod — the workspace concept

Every benchmark run, inside the pod, executes:

```
git clone --no-local /workspace /workspace/.benchmark/<run-id>/repo
git -C /workspace/.benchmark/<run-id>/repo checkout -b ovbench/<run-id>
git -C /workspace/.benchmark/<run-id>/repo submodule update --init --recursive
```

The clone is the AI's workspace. `--no-local` keeps the new `.git` independent — the AI can't accidentally pollute host refs from inside the pod. Submodules are populated so the AI sees the full `plugins/` skill set.

After each iteration, inside the clone:

```
git add -A && git commit --allow-empty -m "iter<k>: score=N, solved=[id1,id2,...]"
```

(Hooks RUN — no `--no-verify`.) At end of run the branch pushes back to the bind-mounted host repo:

```
git push file:///workspace ovbench/<run-id>
```

Post-benchmark on the **host**, `git log ovbench/<run-id>` is the audit trail. `git diff main..ovbench/<run-id>` is the total AI delta. The branch lives in the host's `.git` after the push-back.

Damage isolation: if the AI breaks `layer.yml` parsing or any other invariant, the damage is bounded inside the per-run clone (which is destroyed/garbage-collected between runs). The push-back lands on a dedicated branch that doesn't touch `main`.

### 3. `ov image test --format yaml` is the scorer

After each iteration's nested-podman rebuild into `ghcr.io/overthinkos/<image>:ovbench-<run-id>-iter<k>`, the pod's `ov` shells out:

```
ov image test ghcr.io/overthinkos/<image>:ovbench-<run-id>-iter<k> --format yaml
```

parses the output via `ParseOvTestOutput` in `benchmark_score.go`, and counts scenarios with verdict `solved` as the iteration score. The `Classify` 7-way matrix runs against (baseline, post) ScenarioState pairs computed from the per-run clone's description blocks. Improvements to `ov test` flow through automatically.

### 4. Iteration loop with plateau-driven termination

For each iteration `k` (1-indexed), inside the pod:

1. Refresh scope YAML (still-unsolved scenarios + score-trajectory history).
2. Render prompt with iteration-specific tokens substituted; mirror to `<repo>/.benchmark/{scope.yml,prompt.md}`.
3. Exec the runner directly (no podman-exec round-trip — already in the pod).
4. Nested podman: `ov image build <image> --tag ovbench-<run-id>-iter<k>` from the clone.
5. `ov image test <iter-tag> --format yaml` against the freshly built image.
6. Reload post-iteration descriptions from the clone; classify each pre-AI scenario (7-way verdict). `score_k = count(solved)`.
7. Commit iteration on the `ovbench/<run-id>` branch in the clone.
8. Plateau check: `score_k > best_score` → reset counter; else increment.
9. Exit when plateau counter ≥ `--plateau-iterations` (default 3).

No overall timeout. Per-runner `timeout:` bounds individual AI invocations (default 30m).

## Setup — one-time pod authoring

```bash
# In the project directory:
ov deploy add bench-pod fedora-coder --bind project=$PWD
ov start bench-pod
ov benchmark sync-credentials bench-pod   # one-shot AI-CLI auth sync
```

`fedora-coder` ships every required layer (`ov-full`, `ov-mcp`, `container-nesting`, claude-code, codex, gemini) — no image work needed. `arch-coder` is the Arch sibling.

The pod is **NOT** marked `disposable: true` by default. Persistence preserves the nested-podman build cache (massive iteration speedup) and the AI's auth state across runs. Add `--disposable` only if you want `ov rebuild bench-pod` to wipe state autonomously (typically only for R10 verification per `/ov-dev:disposable`).

For multi-project users: one `bench-pod-<project>` per project, each with its own `--bind project=...`.

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

    Your cwd is the per-run git clone at ${WORKSPACE}. Edits you make are
    captured per-iteration as git commits on branch ovbench/${RUN_ID}.

    MCP tools: ${MCP_ENDPOINT}

    Read scope:              ov benchmark scope
    See prior test output:   ov image test $(ov benchmark last-test-tag) --format yaml
    Exit when done with this iteration.
```

### Substitution tokens

`${PROMPT}`, `${PROMPT_FILE}`, `${WORKSPACE}` (= per-run repo dir), `${TARGET_IMAGE}`, `${TARGET_DEPLOYMENT}`, `${RUN_ID}`, `${ITERATION}`, `${MAX_ITERATIONS}`, `${PLATEAU_ITERATIONS}`, `${PLATEAU_COUNTER}`, `${BEST_SCORE}`, `${MCP_ENDPOINT}` (= `http://localhost:18765/mcp`), `${TAGS}`, `${DEADLINE}`, `${TIMEOUT}`, plus any `${X}` that falls through to `os.Getenv("X")`.

## Supported pod targets

The thin-host model only supports `target: pod` deployments. The old host/vm/k8s dispatch matrix was deleted with the cutover — a pod is what makes the in-pod nested build + test work, and there's no value in maintaining three near-identical paths.

If your host genuinely cannot run pods (rare; both Docker and Podman work), the workaround is to provision a VM that runs nested rootless podman, then run `ov deploy add bench-pod fedora-coder` *inside the VM* — the same pod recipe applies.

**Preflight requirements** on the bench-pod:

- `target: pod` (or default container target).
- Pod is running (`ov status <pod>` shows `Active: active (running)`).
- `/workspace` bind-mount points at the project directory (set with `--bind project=$PWD` at deploy time).
- AI-CLI binary on PATH inside the pod (matches the runner's `command[0]`).
- `ov` binary inside the pod (always present in `fedora-coder`/`arch-coder` via `ov-full`).
- Nested podman works (`ov cmd <pod> -- podman info` succeeds; ships via `container-nesting`).

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

## Credential sync — `ov benchmark sync-credentials <pod>`

A one-shot copy of AI-CLI auth files into the pod's `$HOME` via `podman cp`. Run once at pod creation, again whenever you rotate credentials. The persistent pod retains auth across iterations and across runs — there is no per-run credential sync.

```bash
ov benchmark sync-credentials bench-pod                # all configured runners
ov benchmark sync-credentials bench-pod --runner claude  # filter to one runner
```

Mounts come from `benchmark.runners[*].credentials` in `overthink.yml`. Each is `{src, dst, optional}` where `~` in `dst` resolves against the pod's $HOME (looked up via `getent passwd $(id -u)` inside the pod). Sync the **minimum** file the AI CLI needs to authenticate non-interactively — typically a single refresh-token file. Do not sync whole config directories: `~/.claude` alone is ~430 MB / 7k files, and only `.credentials.json` is load-bearing for `claude -p` auth.

**Concurrency**: a per-pod flock at `/workspace/.benchmark/.lock` prevents two `ov benchmark run` invocations against the same pod from racing on the per-run clone, on nested podman storage, or on the `ovbench/<run-id>` branch namespace. The second concurrent run fails fast.

## AI-facing feedback loop

Each iteration the AI sees a trajectory-aware scope YAML via `ov benchmark scope`:

```yaml
run_id: 20260425-101500-abc123
iteration: 3
max_iterations: 50
plateau_iterations: 3
plateau_counter: 1
best_score: 2
target_image: fedora-coder
history:
  - k: 1
    score: 1
    solved_ids: [desc:layer:sshd:0]
    runtime: 12m14s
  - k: 2
    score: 2
    solved_ids: [desc:layer:sshd:0, desc:layer:ssh-client:0]
    plateau_counter_after: 0
scenarios:
  - id: desc:layer:foo:0
    origin: layer:foo
    baseline_verdict: fail
    pending_steps_current: 2
```

The AI can also run `ov image test $(ov benchmark last-test-tag) --format yaml` to re-inspect the prior iteration's detailed results without triggering a rebuild (the previous iteration's tag is still in the pod's nested storage).

Opt-in expensive self-check: `ov benchmark self-evaluate` rebuilds the current per-run clone into a throwaway tag and runs `ov image test` against it, printing the score. **Pure** — does NOT mutate the active iteration's score. The authoritative score still comes from the harness at end-of-iteration.

## Filesystem layout under `.benchmark/<run-id>/`

Inside the pod (canonical location during a run):

```
/workspace/.benchmark/<run-id>/
├── repo/                        # per-run git clone, branch ovbench/<run-id>
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

After completion, the host's `ov benchmark run` mirrors `/workspace/.benchmark/<run-id>` back to the host's `<projectDir>/.benchmark/<run-id>` via `podman cp`, so `ov benchmark list` / `report` work host-side.

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

## Known follow-ups

- **Deploy-scope scenarios are not yet scored.** `ov image test` runs disposable (no `--include-deploy`), so deploy-scope scenarios — which need a running container — never appear in the post-iteration test output and currently false-trigger as `tampered` or `unchanged`. The fix is to spin up the iter-tag image as a transient `--rm` container inside the pod and run deploy-scope assertions against it. Documented as a follow-up; not bundled into the cutover.

## When to Use This Skill

**MUST be invoked** before authoring runner configs, debugging benchmark loops, interpreting reports, or modifying any `ov/benchmark_*.go` file. Invoke BEFORE reading the source code.

## Cross-references

- `/ov:test` — `ov test` and `ov image test` semantics; the verb catalog the scorer consumes
- `/ov:layer`, `/ov:image` — the YAML surfaces the AI edits
- `/ov-layers:ov-mcp` — the `/workspace` bind-mount convention benchmark relies on
- `/ov-layers:container-nesting` — the rootless-nested-podman recipe that lets the pod build images
- `/ov-images:fedora-coder`, `/ov-images:arch-coder` — the canonical bench-pod images
- `/ov-dev:go` — Go-side implementation map (`benchmark_cmd.go`, `benchmark_runlocal_cmd.go`, `benchmark_synccreds_cmd.go`, `benchmark_loop.go`, `benchmark_clone.go`, `benchmark_score.go`, `benchmark_config.go`)
- `/ov-dev:cutover-policy` — the hard-cutover rules that governed both this and the dispatcher-removal cutover
- `/ov-dev:disposable` — `disposable: true` authorization rules (relevant when running R10 against a disposable bench-pod)
