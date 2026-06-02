---
name: eval
description: |
  MUST be invoked before any work involving: `ov eval` (image / live / run),
  the `eval:` / `deploy_eval:` fields in layer.yml / image.yml / deploy.yml,
  the `org.overthinkos.eval` OCI label, the AI iteration harness loop,
  `kind: ai` / `kind: recipe` / `kind: score` / `kind: eval` (disposable R10
  beds run via `ov eval run <bed>`) in eval.yml, or any declarative
  check authoring. Covers the unified `ov eval` surface: three
  primary modes (image / live / run),
  9 live-container probe verbs (cdp/wl/dbus/vnc/mcp/record/spice/libvirt/k8s),
  verb catalog (file/port/command/http/package/service/process/dns/user/
  group/interface/kernel-param/mount/addr/matching), runtime variable
  resolution (`${HOST_PORT:N}`, `${VOLUME_PATH:name}`, `${CONTAINER_IP}`,
  `${ENV_*}`), deploy.yml overlay rules, authoring gotchas learned the hard
  way (package renames, absent binaries, host vs container network routing),
  AI-iteration loop semantics (plateau-bounded, progressive recipe
  disclosure, watchdog), and the `from:` block for composing recipes from
  existing layer/image/pod/vm tests.
---

# Eval — unified declarative + AI-iteration evaluation

## kind: eval beds + `ov eval run <bed>` — config-driven R10 acceptance

The disposable R10 test beds are **`kind: eval` entities in `eval.yml`**. A
`kind: eval` entity is a deploy-shaped node folded into the Deploy map at load
time, so every deploy verb resolves it by name; `ov eval run <bed>` drives the
full R10 sequence on it:

This is the **ecosystem-wide** rule: every repo-shipped disposable test bed —
the main repo's beds AND every `image/<distro>` submodule's beds (the arch /
cachyos / debian / ubuntu / fedora bootstrap-VM and pacstrap/debootstrap beds) —
is a `kind: eval` entity, in that repo's config (its `overthink.yml` + per-kind sibling files). Repos
ship NO `kind: deploy` test beds. The lone `kind: deploy` exception is an
operator profile, not a test bed (the cachyos submodule's `ov-cachyos`
workstation profile); operator deployments otherwise live in the per-host
`~/.config/ov/deploy.yml`. `disposable: true` is the sole authorization for the
unattended destroy + rebuild on any of them.

A bed is a **candybox** (CLAUDE.md "Candyboxing"): a disposable, secured
container / VM stocked with the FULL toolset — the entire `ov eval` probe surface
(cdp/wl/dbus/vnc/mcp/adb/appium/k8s), nested podman, real package managers, the
whole layer/image library — so the AI can build, deploy, and prove the *real*
composition inside it (the substance of RDD), and `disposable: true` makes
tearing it down and rebuilding fearless. The box is restricted at the boundary,
never by stripping the candy.

1. `ov image build <image>` — build the test artifact (pod beds only).
2. `ov eval image <image>` — image-section + baked layer-section probes.
3. `ov deploy add <bed> <ref>` — apply the bed (or `ov vm create` for vm beds).
4. (pod beds) `ov config <bed>` + `ov start <bed>`.
5. `ov eval live <bed>` — full three-section live probe pass.
6. `ov update <bed>` — fresh-rebuild re-verification (R10 acceptance gate).
7. `ov remove <bed>` (or `ov vm destroy`) — leave the host clean.

**Exclusive-resource preemption wraps the sequence.** When a bed declares
`requires_exclusive: [token...]` (e.g. a GPU-passthrough bed needing the one
physical card), `runEvalBed` acquires a resource-arbitration lease BEFORE step 1
and `defer`-releases it after teardown: any running `preemptible` holder of the
token (e.g. the operator's GPU workstation VM) is gracefully stopped — and the
arbiter waits until it actually powers off so the device is freed — then restored
when the bed finishes (even on failure, for `restore: always`). A holder is never
left stopped: `ov preempt status` / `ov preempt restore` recover a crashed run.
See `/ov-internals:disposable` "The resource-arbitration axis" + `/ov-core:deploy`.

`ov eval run --all-beds` runs every bed name-sorted. Flags: `--keep` (don't
tear down), `--no-rebuild` (skip step 6 — FORBIDDEN for an R10 acceptance run;
needs explicit operator authorization per CLAUDE.md). Per-run logs land in
`.eval/<bed>/<calver>/` (per-step `.log` files + `summary.yml`).

**Run retention (`defaults.keep_eval_runs`).** After `ov eval run` (any path —
bed / `--all-beds` / score), `.eval/<bed|score>/` is trimmed to the newest N run
artifacts (CalVer run dirs, `runs/<id>/` dirs, `result-<calver>.yml`) so harness
output doesn't accumulate unbounded. `NOTES.md` (the durable Syncthing-replicated
memory) is ALWAYS preserved. Set `defaults.keep_eval_runs` in `overthink.yml`
(`0`/absent disables); apply on demand with `ov clean --eval`. See
`/ov-core:clean`.

### The repo's `kind: eval` beds (eval.yml)

| Bed | Target | Ref | Surface |
|---|---|---|---|
| `eval-sway-browser-vnc-pod` | pod | `image: sway-browser-vnc` | cdp/wl/vnc/dbus/mcp/record + pod-side file/service/port/process/http |
| `eval-k3s-vm` | vm | `vm: k3s-vm` | k8s (all 13 methods) + guest-side file/service/port/process, http via port-forward, VmDeployTarget end-to-end |
| `eval-pod` | pod | `image: eval-pod` | combined mechanism bed: `kind: image` build + `kind: layer` composition order + `kind: pod` runtime (nc :18794 + supervisord) + every DeployTarget rendering path |
| `eval-local` | local | `local: eval-local` | `kind: local` layer apply via ShellExecutor |
| `eval-jupyter-pod` | pod | `image: jupyter` | jupyter-mcp regression coverage |
| `eval-jupyter-ml-pod` | pod | `image: jupyter-ml` | jupyter-ml spacy/quarto + GPU MCP probes |
| `eval-versa-pod` | pod | `image: versa` | versa OSM analytics + vector-tile + marimo MCP |
| `eval-android-emulator-pod` | pod | `image: android-emulator` | Android 14 emulator (/dev/kvm) + adb/appium |

Naming: `eval-<descriptor>-<kind>`, dropping a redundant suffix when the
descriptor already equals the kind AND the short form is free (`eval-local`,
`eval-pod`). The harness AI-sandbox pod (the score `pod:` target) is named
`eval-sandbox`, kept disjoint from these beds. Nothing in the `ov` Go code
hardcodes that name — it flows from the score's `pod:` field through
`ResolveScoreTarget`, and prompts reference it via the `${TARGET_NAME}`
substitution token. The supporting
`vm: k3s-vm` + `k8s: vm-k3s-vm` entities live in `eval.yml` alongside the
beds. `disposable: true` is the sole authorization
for the unattended destroy+rebuild (see `/ov-internals:disposable`). Two
load-time guards back the beds: `validateEvalBeds` enforces `target ∈ {pod, vm,
local, android}`, a resolvable cross-ref, and `disposable: true`; `foldEvalBeds`
enforces the name-disjointness from `kind: deploy`. Neither checks host ports —
disjoint ports across beds are the AUTHOR's responsibility (an overlap fails the
second bed at deploy via `CheckPortAvailability`).

### Approximate wall-clock (10-CPU 32-GB reference host)

`eval-pod` ~110s (one build → deploy → eval → fresh-update → teardown cycle
covering all four mechanisms) · `eval-local` ~45s · `eval-k3s-vm` ~5–7 min · the
heavy feature beds (`eval-sway-browser-vnc-pod` ~14 min incl. image build)
longer. **`ov eval run --all-beds` runs beds STRICTLY SEQUENTIALLY (a plain loop
— no concurrency in `ov`), so its wall-clock ≈ the SUM.** To collapse that to ≈
the slowest single bed, parallelize at the AGENT layer — one agent/teammate per
bed, N concurrent `ov eval run <bed>` processes (`/verify-beds` and an agent team
both do this; see `/ov-internals:agents` "Speed levers"). The dominant cost is
the step-1 `ov image build`: a pod bed builds the image ONCE — the "fresh
`ov update`" R10 step is a `systemctl restart` onto the already-built image, not
a second build — and same-base beds share cached layers, so pre-warming a shared
base once makes every sibling bed's build incremental.

**Handling a long-running bed — by mechanism, not by who owns it.** A VM/emulator
bed's `ov eval run` orchestrator runs for minutes-to-tens-of-minutes AND the
libvirt domain / emulator it spawns outlives a single turn. (1) **Launch it as a
harness-tracked background task** (`run_in_background`) — never foreground (the
Bash tool's `timeout`, 120s default / 600s max, kills the call mid-`vm-create`,
orphaning the domain) and
never a sleep/poll loop (the R4 bandaid). (2) **Let the completion notification
drive the next step** — the harness re-invokes the LAUNCHING session when the run
exits, so the launcher must SURVIVE to completion to be notified: the persistent
main session does; an ephemeral sub-agent (returns synchronously) and an idle
teammate (torn down on idle) do NOT, and orphan the bed. EVERY full `ov eval run <bed>`
is a `run_in_background` task owned by a PERSISTENT owner that survives across
turns to be notified — the main session, a background agent, or (interactive
tmux) a split-pane teammate; an in-process teammate CANNOT. Duration-independent;
the 600s is a Bash FOREGROUND cap, irrelevant to a backgrounded bed.
Teammates/sub-agents do bed-local edits + short foreground checks (`ov eval
image`), never the full run. (3) **Reconnect via durable state** —
`.eval/<bed>/<calver>/summary.yml` + the live domain ARE the truth: "done" =
summary.yml present; "alive" = the orchestrator is in the process table; clean up
an orphan (`running` domain, no live orchestrator) with `ov vm destroy <entity>`
before re-running. See `/ov-internals:agents` "The binding rule".

### Prereq for the vm bed

`eval-k3s-vm` depends on the **libvirt user-session daemon**. Enable once:

```bash
systemctl --user enable --now virtqemud.service     # libvirt >= 8 (modular)
# OR (older monolithic):
systemctl --user enable --now libvirtd.service
```

`ov eval run eval-k3s-vm` best-effort starts the unit before `ov vm create`
(via `startLibvirtUserSession()` in `ov/vm.go`); `resolveVmBackend()` surfaces
a clear error when libvirt is absent. The `k3s-vm:` vm template pins
`backend: libvirt` explicitly so the silent `auto -> qemu` fallback can't mask
a missing daemon. See `/ov-vm:vm` "Prereq", `/ov-eval:libvirt`.

### Why this is config-driven

Beds are `kind: eval` config; the runner reads the bed node directly. This
means the runner works for ANY bed an operator defines — `ov eval run <name>`
resolves the bed by name through the same path every deploy verb uses, with no
hardcoded bed table to keep in sync.

## The 10 Testing Standards (READ FIRST — referenced by CLAUDE.md R1–R10)

An earlier agent claimed "cutover complete, all tests pass" after green `go test ./...` runs, then built an image that crash-looped at startup because a Containerfile stage was silently dropped. The unit tests didn't notice — they only exercised YAML loaders. **Unit tests are NOT a substitute for running the service.**

These are the 10 standards referenced in CLAUDE.md's AI attribution tier ("fully tested and validated"). Each is keyed to a CLAUDE.md R-rule. Apply them whenever a change could affect Containerfile generation, OCI labels, init systems, service startup, or deploy code.

0. **Prove every HIGH-RISK assumption BEFORE you edit (RDD — Risk Driven Development)** — the proactive bookend to Standard 10's fresh-rebuild gate. Low-risk orientation ("what does layer X do") is a skill lookup (R0, zero risk); every high-risk assumption — including any a skill or the code merely *asserts*, and above all whether this layer composition at its latest available versions builds / deploys / runs TOGETHER — is proven on a `disposable: true` bed FIRST (`ov eval` it). Never accept docs or code as ground truth for a high-risk decision; if the bed disagrees with a skill, the skill is stale — fix it. Standard 0 (validate forward, riskiest-first) and Standard 10 (re-verify on a fresh rebuild) are the two ends of the same loop.

1. **Build a real artifact** (R1) — `ov image build <image>` / `go build` / `ov vm build <vm>`. Not just `go test`. Not just `ov image generate`.
2. **Verify the emitted artifact's content** (R3) — `grep -c supervisord-conf .build/<image>/Containerfile` for any image that uses supervisord; `virsh dumpxml` for a VM; `podman inspect --format '{{.Created}}'` to confirm the image was just rebuilt.
3. **Verify critical OCI / capability labels post-build** (R4) — `podman inspect <ref> --format '{{index .Config.Labels "org.overthinkos.init"}}'` returns the expected value. Empty / missing → the detection path silently returned nil → regression.
4. **Deploy to a DISPOSABLE target** (R10) — NEVER experiment on a resource that doesn't carry `disposable: true`. If no suitable disposable target exists, create one first (`ov deploy add <name> <ref> --disposable` or mark a VM in vm.yml and `ov vm create`). The setup is part of the task. On a disposable target: `ov update <name>` (unattended). On anything else: confirm with the user before any destroy.
5. **Target must reach steady-state** — `systemctl --user status ov-<image>.service` → `Active: active (running)`; `virsh domstate <vm>` → `running (booted)`; SPICE socket file exists and accepts a handshake. If `start-limit-hit` appears, the container is crashing — reproduce directly via `podman run --rm <image> <entrypoint>`.
6. **Run the declarative test suite** — `ov eval live <image>` full three-section pass against the live container (or `--uri` / `--host` remote equivalent for a remote target).
7. **Verify the deployed binary is the one you built** (R8) — `ov version` on the target matches the expected CalVer; `podman inspect <ref> --format '{{.Created}}'` timestamp is from THIS build, not the prior one. Source-only changes (Syncthing, git push) do NOT update the deployed binary; you must build AND deploy on the target host.
8. **Verify runtime deps are installed via package management** (R9) — `which nc`, `rpm -q <pkg>`, `pacman -Q <pkg>`. Manual installs do NOT count — they won't survive a fresh install on a synced host. Every runtime dep must live in `setup.sh` + `pkg/arch/PKGBUILD`.
9. **Leave the target healthy, not paused/errored** — final snapshot of `virsh domstate` / `systemctl status` / `podman ps` is healthy. If the target is in a broken state during exploration, `ov update` it back to the committed config before continuing — never layer experiments on broken state.
10. **Re-verify on a FRESH rebuild after committing the source-level fix** (R10) — `ov update <disposable-target>` one more time from clean, with the new source applied. Run standards 1–9 again against this fresh rebuild. **THIS IS THE ACCEPTANCE GATE.** A fix that works on a hand-patched target but not on a fresh rebuild is a regression waiting for the next unrelated rebuild to wipe your patch. Paste BOTH the exploratory-pass output AND the fresh-rebuild-pass output into the conversation — the user sees both.

### `ov eval live parent.child` reaches the actual leaf

`ov eval live <parent>.<child>` walks the dotted deployment path through
`ResolveDeployChain` (`ov/deploy_chain.go`) and constructs a multi-hop
`DeployExecutor` chain that lands probes inside the leaf's actual
venue — `command: id` for a pod-in-VM leaf returns the inner pod's user, not
the parent VM's. Live-eval and AI-iteration-scoring chain construction go
through the same code path `ov deploy add` uses.
For deeply-nested paths, each segment adds one hop:

```bash
ov eval live eval-vm                            # → SSHExecutor (1 hop)
ov eval live eval-vm.inner                      # → SSH + podman exec ov-eval-vm_inner (2 hops)
ov eval live eval-vm.inner.deeper               # → SSH + 2× podman exec (3 hops)
```

Same chain primitive (`NestedExecutor`) used everywhere — `ov deploy
add`, `ov eval live`, `ov eval run` (AI iteration scoring). See "ONE
primitive: scenario.pod (flat or dotted)" below for the symmetric
authoring surface in eval recipes (`kind: recipe`).

### Specific anti-patterns observed and banned

- **"Unit tests pass → cutover done"** — no. Build + deploy + run + test, every time.
- **"Retested after update → still passing!"** but the pre-update test was against the old running image and the post-update container failed to come up — see the `is not running` error and conclude the update broke it, not that "tests pass in aggregate."
- **"Service start failed, probably a transient"** — no. `A dependency job for X failed` + immediate exit is a real error. Read the service log: `podman run --rm <image> <entrypoint>` will reproduce the failure directly.
- **"Lifecycle: dev implies disposable"** — no. `disposable: true` is the ONE authorization for autonomous destroy + rebuild. Lifecycle tags are human-facing metadata; they do not authorize anything. See `/ov-internals:disposable`.
- **"This is a dev box so I can just nuke it"** — no. The only authorization for autonomous destroy is the explicit `disposable: true` field on a specific deploy. Everything else requires user confirmation, regardless of hostname.
- **"I tested on the VM I've been patching all afternoon, looks fine"** — incomplete. Run `ov update <disposable-target>` once more from clean and re-verify before claiming success. Without the fresh-rebuild re-verification, your "fix" may be latent on hand-patched state.
- **"I'll test it later / Phase 2"** — no. If the plan said "clean cutover in one PR", don't invent a Phase 2.
- **"The skill/code says so, so I don't need to test it"** — for a HIGH-RISK assumption, no: docs drift and code has bugs; prove it on a disposable bed FIRST (Standard 0). Conversely, do NOT bed-test a LOW-risk orientation fact a skill already states — that's wasted effort. Risk, not documentation status, is the trigger.

If the container needs state that's only available in deploy (volumes, env, tunnel), author the test at `scope: deploy`. If it needs something at build only (binary path, package presence), author at `scope: build`. Both scopes must pass for the cutover to be real.

**Confidence tier mapping:** The "fully tested and validated" confidence level in CLAUDE.md's AI-attribution table requires ALL 10 standards met — including Standard 10, the fresh-rebuild re-verification. Anything short of that ships at a lower confidence tier.

## Overview

`ov` ships a goss-inspired declarative testing framework built into the
CLI. Eval checks are authored inline under `eval:` (or `deploy_eval:`) in
`layer.yml`, `image.yml`, or `deploy.yml`. They are **embedded as a
three-section OCI label** (`org.overthinkos.eval` → `{layer, image, deploy}`)
so any pulled image is self-testable without its source repo. A local
`deploy.yml` overlay can add checks or override baked ones by `id:`.

The runner resolves **deploy-time variables** (actual host port mappings,
volume backings, env vars, container IP, DNS) at execution time, so a
check written once works unchanged when `deploy.yml` remaps ports.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Pure-image eval (disposable, build-scope) | `ov eval image <image>` | Layer + image sections only, in `podman run --rm` (no host port mappings, no volumes attached) |
| Live full-stack eval (running deployment) | `ov eval live <name> [-i instance]` | All three sections run via `podman exec` / SSH / nested chain, with full runtime variable resolution |
| R10 bed (full sequence) | `ov eval run <bed>` / `ov eval run --all-beds` | Build → eval image → deploy → eval live → fresh update → tear down on a `kind: eval` disposable bed. Canonical R10 gate |
| AI iteration loop | `ov eval run <score>` | Drives an AI through plateau-bounded iterations against `kind: score` |
| Validate authored tests at config time | `ov image validate` | Schema, scope/variable consistency, id uniqueness |
| Inspect effective spec | `ov image inspect <image>` | JSON includes merged eval structure |
| Filter by verb | `ov eval live <name> --filter file --filter port` | Repeatable |
| Filter by section | `ov eval live <name> --section deploy` | One of: layer / image / deploy |
| Output format | `ov eval live <name> --format json\|tap\|text` | Default text |

## Exit codes

`ov eval` uses a goss/pytest-style three-way exit convention so automation
(and `ov eval run <bed>`) can tell "the thing under test is broken" apart from
"the eval couldn't run":

| Code | Meaning |
|---|---|
| `0` | All checks passed. |
| `1` | Command / usage / infra error — bad args, undeclared deploy, container not running, image build/deploy/vm-create failed. The eval never produced a pass/fail verdict. |
| `2` | The eval RAN and one or more **checks FAILED** (`EvalCheckFailExitCode`). |

`ov eval image` / `ov eval live` return `2` directly when checks fail.
`ov eval run <bed>` propagates `2` when the bed's **eval step** (eval-image /
eval-live) fails on checks, but `1` when an **infra step** (build / deploy /
vm-create) fails — so a broken bed image is distinguishable from a genuine
test failure. `ov eval run --all-beds` returns `2` only when *every* failing
bed failed on checks; any infra failure makes it `1`. Implementation:
`EvalFailedError` + `EvalCheckFailExitCode` (ov/eval_cmd.go), mapped to the
process exit code in `main()` via `errors.As`.

## Image preflight (host-target runs only)

When `ov eval run --on-host <name>` (or any score whose target
resolves to `TargetKindHost`) dispatches the runner, an image
preflight runs FIRST: walk the score's recipes, collect every
distinct `scenario.pod` plus `score.target_image`, deduplicate, and
ensure each image is present in local podman storage BEFORE handing
off to the host runner.

For each discovered image, the algorithm is:

1. `LocalImageExists(podman, ref)` — short-circuit if already
   present. Idempotent on re-runs.
2. `ov image pull <ref>` — preferred path. Resolves short names via
   `cfg.Images[<name>]`, full registry refs pass-through, remote
   `@github.com/...` refs walk through `ResolveRemoteImage`.
3. `ov image build <name>` — fallback when pull fails AND the
   identifier is a short name resolvable via the project's
   `image.yml`. Build fallback is local-only; for any non-short
   identifier that fails to pull, the preflight aborts with an
   actionable error.

Failures abort the eval BEFORE any scenario runs, so operators see
problems early rather than mid-scenario.

This is the **only** image-fetch surface in the system: deploys (any
target — `local`, `pod`, `vm`, `k8s`) emit zero image-pull steps. A
`kind: local` template has no `image:` field; image preflight lives on this
verb. Operators with legacy YAML run `ov migrate`. See
`/ov-local:local-spec` "What the deploy does NOT do" and CLAUDE.md
"Deploy fetches NOTHING speculative".

Lives in `ov/eval_image_preflight.go`
(`EnsureImagePresent`, `ensureScoreImages`); wired into
`EvalRunCmd.Run` for the `case TargetKindHost:` arm. Pod / VM / k8s
targets carry their own image inside their respective deploy schema
and never trigger the preflight.

## Three primary modes (`ov eval image` / `live` / `run`)

The surface is three orthogonal verbs, each named for what it evaluates:

- **`ov eval image <image>`** — evaluates the image **artifact** in
  isolation. Spawns a disposable container via `podman run --rm
  <image-ref>` (`ImageExecutor`). Only `eval:`-block checks at
  `scope: build` run; deploy-scope checks are skipped with a clear
  message. The right choice for build-time invariants ("did this
  binary land at this path?", "is this package installed?") where a
  running service would only add noise.

- **`ov eval live <name>`** — evaluates a **running deployment**
  (pod / vm / host / k8s; auto-resolved by `<name>` against
  deploy.yml). Uses `ContainerExecutor` / `SSHExecutor` /
  `NestedExecutor` (for dotted-path children) + `ResolveEvalVarsRuntime`
  so deploy-scope checks see real supervisord state, real
  `HOST_PORT:<N>` mappings (including host-networked containers via
  `HostConfig.NetworkMode` detection), and real in-container env.
  Same dispatcher that powers `ov eval live parent.child` for
  pod-in-vm topologies.

- **`ov eval run <score>`** — drives an AI runner through
  plateau-bounded iterations against a `kind: score`. See "AI
  iteration loop semantics" below.

The mode is **explicit in the verb**; there is no autodetect or
implicit fallback. Choose the mode by picking the right verb.

The banner after `ov eval image` reports the image ref:

```
Image: ghcr.io/overthinkos/fedora-coder:latest
```

The `meta.Image` short-name (from the `org.overthinkos.image` OCI
label) is used by `ov eval live` for the `ov-<image>` container-name
lookup — full image refs like `ghcr.io/overthinkos/fedora-coder:latest`
are correctly mapped to `ov-fedora-coder`. Implementation:
`ov/eval_cmd.go` `EvalImageCmd.Run()` and `EvalLiveCmd.Run()`.

## Subcommands: live-container verbs

`ov eval live` is both the declarative test runner AND the grouping point for the
interactive verbs that drive a running service. Kong's `default:"withargs"`
tag means `ov eval live <image>` still dispatches to the runner — only the
explicit subcommand names below take over when matched.

| Subcommand | Skill | Purpose |
|-----------|-------|---------|
| `ov eval cdp …` | `/ov-eval:cdp` | Chrome DevTools Protocol — open/list/close tabs, click, type, eval, screenshot, axtree |
| `ov eval wl …` | `/ov-eval:wl` | Wayland desktop input/windows/clipboard + sway IPC |
| `ov eval dbus …` | `/ov-eval:dbus` | D-Bus calls, introspection, desktop notifications |
| `ov eval vnc …` | `/ov-eval:vnc` | VNC framebuffer screenshot + click/key/type/passwd |
| `ov eval mcp …` | (this skill) | MCP client — ping/list-tools/list-resources/list-prompts/call/read/servers against any `mcp_provides` endpoint. Speaks `github.com/modelcontextprotocol/go-sdk` (Streamable HTTP by default, SSE when `transport: sse`). |
| `ov eval record …` | `/ov-eval:record` | Recording sessions — start/stop/list/cmd. Terminal (asciinema) or desktop (pixelflux/wf-recorder). Container-only. |
| `ov eval spice …` | `/ov-eval:spice` | SPICE wire client for VMs — handshake, native-SPICE framebuffer screenshot, input injection. VM-only. |
| `ov eval libvirt …` | `/ov-eval:libvirt` | libvirt-RPC test commands for VMs — info, domain XML, QMP, qemu-guest-agent, snapshots, events. VM-only. |

These eight verbs live under `ov eval` because every one of them is a "probe or
drive a running service" operation — the same surface the declarative test
runner composes when it executes checks.

**Reserved image names:** because subcommand names take priority when
matched, an image literally named `cdp`, `wl`, `dbus`, `vnc`, `mcp`,
`record`, `spice`, or `libvirt` cannot be run via `ov eval live <name>` —
use the explicit `ov eval live <name>` form or rename the image. No such
images currently exist in `image.yml`.

**Gotcha — stale container-baked `ov` binary:** `ov eval dbus notify` and
`ov eval dbus call` delegate to the container's own `ov` binary (see
`ov/notify.go:20`, `ov/dbus.go:195,229`). If the container bakes an `ov`
binary too old to know the `ov eval <verb>` subcommand path, the delegation
fails. Fix by rebuilding and redeploying any image that bakes `ov` (grep
`image.yml` for `- ov$` to find them). Test runner itself is unaffected —
this only bites the host→container delegation paths.

## Authoring: the `eval:` list

Every check is a **list entry with exactly one verb discriminator** plus
shared modifiers and verb-specific attributes. This mirrors the `task:`
pattern in `layer.yml`.

### Gold-standard pattern (redis layer)

```yaml
# layers/redis/layer.yml
eval:
  # Build-scope — run inside the built image via `podman run --rm`.
  - id: redis-binary
    file: /usr/bin/redis-server
    exists: true
  - id: redis-cli-binary
    file: /usr/bin/redis-cli
    exists: true
  - id: redis-package
    package: valkey-compat-redis     # see "Authoring Gotchas" on Fedora 43 renames
    installed: true

  # Deploy-scope — HOST_PORT:N resolves to the effective port mapping, so
  # the same test works whether the deploy publishes 6379:6379 or 16379:6379.
  - id: redis-responds
    scope: deploy
    command: redis-cli -h 127.0.0.1 -p ${HOST_PORT:6379} ping
    stdout: PONG
    in_container: false    # run redis-cli from the host
  - id: redis-port-open
    scope: deploy
    addr: 127.0.0.1:${HOST_PORT:6379}
    reachable: true
```

The six entries cover: binary existence, package identity, a functional
live probe, and raw TCP reachability. Adopt this shape for any primary
service layer.

### Cross-distro package names (`package_map:`)

The `package:` verb chains `rpm -q || dpkg -s || pacman -Q` with a single
literal name, so a test that works on Fedora (`openssh-server`) fails on
Arch (where the package is just `openssh`). `package_map:` is the
authoring hook: the first key that matches any of the image's
`distro:` tags wins; otherwise the `package:` scalar is used as the
fallback. Source: `ov/testrun_verbs.go:resolvePackageName` + the
`Runner.Distros` field wired from `meta.Distro` at both test entry
points.

```yaml
# layers/sshd/layer.yml — cross-distro authoring
- id: openssh-server-package
  package: openssh-server              # Fedora / Debian default
  package_map:
    arch: openssh                 # Arch ships the metapackage as 'openssh'
    fedora: openssh-server
    fedora:43: openssh-server          # explicit version tag — matches before 'fedora'
  installed: true
```

Tag priority: entries in `distro:` are in priority order (e.g.
`["fedora:43", fedora]`), so `package_map:` can be keyed on either the
version-specific or the family tag, and the more-specific tag wins
naturally. An empty-string map value falls through to the next tag
(see `TestResolvePackageName` in `ov/testrun_verbs_test.go`).

### Verb catalog (goss-parity feature set)

| Verb | Typical attributes | Notes |
|------|---------------------|-------|
| `file` | `exists`, `mode`, `owner`, `group_of`, `filetype`, `contains`, `sha256` | `group_of` (not `group`) avoids the verb collision |
| `package` | `installed`, `versions` | Tries rpm / dpkg / pacman — exact match on installed name (see gotchas) |
| `service` | `running`, `enabled` | Supervisord first, then systemd |
| `port` | `listening`, `ip`, `reachable` | **`listening` needs `ss`/`netstat` inside the container** — absent from minimal images |
| `process` | `running` | **Needs `pgrep`** — absent from minimal images |
| `command` | `exit_status`, `stdout`, `stderr`, `in_container` | Default runs via `podman exec`; set `in_container: false` to run from host |
| `http` | `status` (int, single value), `body`, `headers`, `method`, `request_body`, `allow_insecure`, `no_follow_redirects`, `ca_file`, `timeout` | Host-side under `ov eval live`; curl-in-container under `ov eval image` |
| `dns` | `resolvable`, `addrs`, `server` | Host resolver for `ov eval live`; `getent hosts` for `ov eval image` |
| `user` | `uid`, `gid`, `home`, `shell` | `getent passwd` |
| `group` | `gid`, `groups` | `getent group` |
| `interface` | `mtu`, `addrs` | `ip -o addr show` |
| `kernel-param` | `value` (scalar or matcher) | `sysctl -n` |
| `mount` | `mount_source`, `filesystem`, `opts` | `findmnt` |
| `addr` | `reachable`, `timeout` | Pure Go-native `net.DialTimeout` for `ov eval live`; `nc` for `ov eval image` |
| `matching` | `contains` | Pure in-process value matching — no target probe |
| `cdp` | Method name (status/list/eval/text/html/url/axtree/screenshot/open/click/type/raw/wait/coords + spa-*) + method-specific modifiers (`tab`, `expression`, `url`, `selector`, `text`, `artifact`, `x`, `y`) + shared `stdout`/`stderr`/`exit_status`/`artifact_min_bytes` | **Deploy-scope only.** Wraps `ov eval cdp <method>`. See Live-container verb catalog below. |
| `wl` | Method name (screenshot/status/click/type/key/key-combo/mouse/scroll/drag/clipboard/toplevel/windows/focus/close/geometry/xprop/atspi/exec/resolution + overlay-*/sway-*) + method-specific modifiers (`x`, `y`, `text`, `key`, `combo`, `target`, `action`, `artifact`) + shared matchers | **Deploy-scope only.** Wraps `ov eval wl <method>` (including `sway` and `overlay` nested subgroups). See Live-container verb catalog below. |
| `dbus` | Method name (list/call/introspect/notify) + method-specific modifiers (`dest`, `path`, `method`, `args`, `text`) + shared matchers | **Deploy-scope only.** Wraps `ov eval dbus <method>`. |
| `vnc` | Method name (status/screenshot/click/mouse/type/key/rfb/passwd) + method-specific modifiers (`x`, `y`, `text`, `key`, `artifact`) + shared matchers | **Deploy-scope only.** Wraps `ov eval vnc <method>`. |
| `mcp` | Method name (ping/servers/list-tools/list-resources/list-prompts/call/read) + method-specific modifiers (`tool`, `uri`, `input`, `mcp_name`) + shared matchers | **Deploy-scope only.** Speaks `github.com/modelcontextprotocol/go-sdk` to any `mcp_provides` endpoint. See "Method allowlist — mcp" below. |

### Shared modifiers

| Field | Purpose |
|-------|---------|
| `id` | Optional stable identifier. Enables `deploy.yml` to override by `id`. Unique per section per image. |
| `description` | Human-readable label for reports. |
| `skip: true` | Always skip this check (reported but doesn't fail the run). |
| `exclude_distros: [<tag>, ...]` | Skip the check when any of the image's `distro:` tags matches an entry. Use for probes that only apply on some distros (e.g. `file: /usr/bin/fastfetch` is valid on Fedora/Arch/Debian but fastfetch is dropped from Ubuntu 24.04's noble main). Matched against the image's full distro list (`["ubuntu:24.04", "ubuntu", "debian"]`), so either `ubuntu:24.04` or `ubuntu` matches. See `ov/testspec.go:Check.ExcludeDistros` and `ov/testrun.go:runOne`. |
| `timeout: "5s"` | Per-check timeout (http, addr). |
| `scope: build\|deploy` | Default `build` at layer/image level, `deploy` at deploy level. |

#### `exclude_distros:` worked example

```yaml
- id: fastfetch-binary
  file: /usr/bin/fastfetch
  exists: true
  exclude_distros:
    - ubuntu:24.04
```

On `ghcr.io/overthinkos/ubuntu-coder:latest` this reports `skipped (excluded on distro "ubuntu:24.04")` instead of `failed`. On any other image it runs normally. Prefer this over dropping the test entirely — it keeps the guard in place for Fedora/Arch/Debian and documents why Ubuntu is special.

**Compared to `package_map:`** — `package_map:` changes what the test *probes for* per distro (same semantic, different package name). `exclude_distros:` skips the check entirely — use it when the functionality genuinely doesn't exist on a given distro.

### Matcher forms

Any `MatcherList` attribute (`stdout`, `stderr`, `body`, `headers`,
`contains`, `opts`, `value`) accepts three shapes — picked by yaml.v3
at parse time:

```yaml
stdout: PONG                         # scalar  → [{equals: PONG}]
stdout: [PONG, READY]                # list    → [{equals: PONG}, {equals: READY}]
stdout:
  - equals: PONG                     # operator map
  - contains: ["ready", "ok"]
  - matches: "^[A-Z]+$"
  - not_contains: "error"
```

Supported operators: `equals`, `not_equals`, `contains`, `not_contains`,
`matches`, `not_matches`, plus `lt`/`le`/`gt`/`ge` (numeric, added as
verbs need them).

**`status:` is NOT a MatcherList** — it's a plain `int` on the `http`
verb. One code per test. See Authoring Gotcha #2.

### Live-container verb catalog: `cdp`, `wl`, `dbus`, `vnc`, `mcp`

These five verbs wrap the corresponding `ov eval <verb> <method>` CLI
subcommands so every live-container operation (browser automation, Wayland
input/screenshot, D-Bus calls, VNC framebuffer capture, MCP protocol
probes) is authorable as a declarative check. All five are **deploy-scope
only** — they need a running container with port mappings; `ov image
validate` rejects them in build scope, and `ov eval image` skips them at
runtime with a clear message.

**Assertion semantics:** subprocess delegation — the runner executes
`ov eval <verb> <method> <image> <args…> [-i <instance>]` on the host,
captures stdout/stderr/exit, and feeds the output through the existing
matcher pipeline. Queries (status/list/eval/text/html/url/axtree/screenshot/…)
produce assertable output. Side-effect actions (click/type/open/close/…) pass
when they exit 0 — follow them with a query check to verify the effect.

**No state chaining (Phase 1):** there's no framework-managed variable to
carry a tab ID or window handle from one check to the next. Authors rely
on conventions (new tabs are usually ID 1) and capture state manually via
follow-up `cdp: list` checks if needed. Phase 2 (deferred) will add
`capture:` + `${CAPTURED:name}` for stateful flows.

#### Method allowlist — `cdp` (15 methods + 6 SPA-nested)

Queries: `status`, `list`, `url`, `text`, `html`, `eval`, `axtree`,
`coords`, `raw`, `wait`, `screenshot`.
Actions: `open`, `close`, `click`, `type`.
SPA-nested (selkies coordinate-scaling / passthrough input):
`spa-status`, `spa-click`, `spa-type`, `spa-key`, `spa-key-combo`,
`spa-mouse`.

```yaml
- id: cdp-up
  cdp: status
  stdout:
    equals: ok

- id: cdp-page-title
  cdp: eval
  tab: "1"
  expression: "document.title"
  stdout: "Dashboard"

- id: cdp-screenshot-valid
  cdp: screenshot
  tab: "1"
  artifact: /tmp/cdp.png
  artifact_min_bytes: 10000       # PNG must be non-empty
```

#### Method allowlist — `wl` (22 top-level + 4 overlay + 12 sway)

Queries: `screenshot`, `status`, `toplevel`, `windows`, `geometry`,
`xprop`, `atspi`, `clipboard`.
Actions: `click`, `double-click`, `mouse`, `scroll`, `drag`, `type`,
`key`, `key-combo`, `focus`, `close`, `fullscreen`, `minimize`, `exec`,
`resolution`.
Overlay-nested: `overlay-list`, `overlay-status`, `overlay-show`,
`overlay-hide`.
Sway-nested: `sway-tree`, `sway-workspaces`, `sway-outputs`, `sway-msg`,
`sway-focus`, `sway-move`, `sway-resize`, `sway-kill`, `sway-floating`,
`sway-layout`, `sway-workspace`, `sway-reload`.

```yaml
- id: wl-desktop-captured
  wl: screenshot
  artifact: /tmp/wl.png
  artifact_min_bytes: 10000

- id: wl-has-windows
  wl: toplevel
  stdout:
    matches: "."        # at least one line of output

- id: wl-sway-has-workspace-1
  wl: sway-workspaces
  stdout:
    contains: '"name":"1"'
```

#### Method allowlist — `dbus` (4 methods)

Queries: `list`, `call`, `introspect`.
Actions: `notify`.

```yaml
- id: dbus-notifications-registered
  dbus: list
  stdout:
    contains: "org.freedesktop.Notifications"

- id: dbus-get-capabilities
  dbus: call
  dest: org.freedesktop.Notifications
  path: /org/freedesktop/Notifications
  method: org.freedesktop.Notifications.GetCapabilities
  args: []
  stdout:
    contains: body
```

#### Method allowlist — `vnc` (8 methods)

Queries: `status`, `screenshot`, `rfb`.
Actions: `click`, `mouse`, `type`, `key`, `passwd`.

```yaml
- id: vnc-up
  vnc: status
  stdout:
    equals: ok

- id: vnc-framebuffer-captured
  vnc: screenshot
  artifact: /tmp/vnc.png
  artifact_min_bytes: 5000
```

#### Method allowlist — `mcp` (7 methods)

Queries: `ping`, `servers`, `list-tools`, `list-resources`, `list-prompts`,
`read`.
Actions: `call`.

The `mcp` verb speaks the Model Context Protocol to any server advertised
via `mcp_provides`. URL resolution is automatic — the runner reads the
image's `org.overthinkos.mcp_provides` OCI label, substitutes
`{{.ContainerName}}`, runs the entries through `podAwareMCPProvides`, and
then rewrites the host portion to `127.0.0.1:<published-host-port>` using
the same port-mapping data that powers `${HOST_PORT:N}`. No URL argument
is needed in YAML.

Transport dispatch: `transport: http` (or empty) → Streamable HTTP;
`transport: sse` → SSE. Anything else is rejected at dial time.

Disambiguation: if an image declares multiple `mcp_provides` entries,
add `mcp_name: <server>` on the check; otherwise the single entry
auto-picks.

```yaml
- id: mcp-ping
  scope: deploy
  mcp: ping
  timeout: 10s                # optional; default 30s

- id: mcp-list-tools
  scope: deploy
  mcp: list-tools
  stdout:
    - contains: insert_cell   # plaintext "name\tdescription" per line
    - contains: execute_cell

- id: mcp-call-tool
  scope: deploy
  mcp: call
  tool: list_notebooks        # required
  input: "{}"                 # optional JSON arg blob
  exit_status: 0              # assert no IsError

- id: mcp-read-resource
  scope: deploy
  mcp: read
  uri: file:///tmp/example.txt
  stdout:
    - matches: "."            # non-empty body
```

Ad-hoc CLI equivalents (each leaf also supports `--json` for programmatic
use and `--name <server>` for multi-server images):

```bash
ov eval mcp ping       <image> [-i <instance>] [--name <server>]
ov eval mcp servers    <image> [-i <instance>]
ov eval mcp list-tools <image> [-i <instance>] [--name <server>]
ov eval mcp call       <image> <tool> '<json-args>' [-i <instance>]
ov eval mcp read       <image> <uri>            [-i <instance>]
```

Output format: tab-separated plaintext (one record per line) so matchers
can `contains:` without JSON decoding. `list-tools` emits
`<name>\t<description>`; `list-resources` emits `<uri>\t<name>\t<mime>`;
`call` emits the concatenated `TextContent` payloads; `ping` emits `ok`.

#### Artifact-validation modifiers

For artifact-producing methods (`cdp: screenshot`, `wl: screenshot`,
`vnc: screenshot`, `libvirt: screenshot`, `spice: screenshot`,
`record: stop`), set `artifact: <path>` to tell `ov` where to write the
output AND any combination of the modifiers below to assert the
artifact's correctness post-run.

- **`artifact_min_bytes: <N>`** — assert the file is at least N bytes
  after the run. Guards against zero-byte files. Cheap (`os.Stat`
  only).
- **`artifact_min_dimensions: WxH`** — assert decoded image width and
  height are each at least the given values. Reads only the PNG/JPEG
  header via `image.DecodeConfig`, so essentially free. Catches "the
  screenshot ran but the compositor produced 320×240 instead of the
  expected 1920×1080".
- **`artifact_not_uniform: true`** — assert the image is not uniformly
  one color. Decodes the full image and samples 100 pixels at
  deterministic stride positions; fails if every sampled pixel has
  the same RGBA. Catches the failure mode `artifact_min_bytes` is
  blind to: a 100KB all-black PNG passes the byte check but fails
  this one. Use on every screenshot probe where "the image has real
  content" matters.
- **`artifact_min_cast_events: <N>`** — for asciinema `.cast`
  artifacts (e.g. `record: stop` in terminal mode): validates the
  first line is the asciinema v2 header and counts at least N event
  lines after it. Catches "the recording started and stopped but
  captured nothing because nothing was typed".

```yaml
# Combined screenshot validity assertion — bytes + dimensions + content.
- id: cdp-screenshot-real
  cdp: screenshot
  tab: "1"
  artifact: /tmp/cdp.png
  artifact_min_bytes: 5000
  artifact_min_dimensions: 800x600
  artifact_not_uniform: true

# Recording validity — bytes + event count.
- id: record-cast-has-events
  record: stop
  artifact: /tmp/session.cast
  artifact_min_bytes: 200
  artifact_min_cast_events: 5
```

The four modifiers are independent — set just the one you need or
combine all four for the strongest "the artifact is real" assertion.
Failure messages identify the specific artifact and the specific
threshold that wasn't met.

#### Gotcha — stale container-baked `ov` binary

The `dbus:` verb invokes the container's `ov` binary via delegation. If
the container bakes an `ov` binary too old to know the `ov eval <verb>`
subcommand path, the delegation fails. Rebuild and redeploy any image that
bakes `ov` (grep `image.yml` for `- ov$`). The test runner itself is
unaffected; this only bites the host→container delegation paths in the
`dbus` verb.

## Authoring Gotchas (learned the hard way)

These are the non-obvious issues that surface only when you run the tests
against real containers. Skim before writing new checks.

### 1. Host-side deploy tests: `127.0.0.1:${HOST_PORT:N}`, NOT `${CONTAINER_IP}:${HOST_PORT:N}`

In rootless podman, the container's pod-network IP (e.g. `10.89.0.x`) is
not routable from the host. `${HOST_PORT:N}` resolves to a port on the
host's loopback that forwards into the container. Combining them is
nonsense:

```yaml
# ❌ WRONG — 10.89.0.x:HOST_PORT is unreachable from either side.
addr: ${CONTAINER_IP}:${HOST_PORT:8080}

# ✓ RIGHT — host-side access via the forwarded port.
addr: 127.0.0.1:${HOST_PORT:8080}

# ✓ ALSO RIGHT — in-container, using the container port directly.
command: curl -fsS http://127.0.0.1:8080/health
in_container: true
```

Use `${CONTAINER_IP}` only from inside another container in the same pod.

### 2. `http` status is a single int — no lists

```yaml
# ❌ WRONG — yaml unmarshals a list to int fails with "cannot unmarshal !!seq"
status: [200, 301, 302]

# ✓ RIGHT — pick the most-likely code. Or author multiple tests.
status: 200
```

For endpoints that return 400/401 when probed without auth (MCP servers,
API endpoints), `status: 400` is a legitimate "endpoint exists and refuses
empty input" check.

### 3. `pgrep`, `ss`, `nc` are NOT in minimal images

The `process:`, `port: listening`, and `nc`-based command tests silently
fail on images that skip `procps-ng` / `iproute` / `ncat`. Portable
alternatives:

| Intent | Portable alternative |
|--------|----------------------|
| "supervisord is running" | `command: supervisorctl pid; exit_status: 0` (in_container) |
| "some program under supervisord" | `service: <name>; running: true` |
| "port N listening in-container" | `command: curl -fsS http://127.0.0.1:N/<route>; in_container: true` |
| "port reachable from host" | `addr: 127.0.0.1:${HOST_PORT:N}; reachable: true` (Go-native dial) |

### 4. `supervisorctl status` exits 3 when ANY program is non-RUNNING

Layers like `hermes` declare `autostart=false` programs (e.g.
`hermes-whatsapp`), which leaves them STOPPED. `supervisorctl status`
returns exit 3 — fails `exit_status: 0` checks.

```yaml
# ❌ BRITTLE — fails if any program is autostart=false.
command: supervisorctl status
exit_status: 0

# ✓ ROBUST — `pid` asks for supervisord's own PID, 0 iff the socket
# responds.
command: supervisorctl pid
exit_status: 0
in_container: true
```

### 5. Know which stream a `--version`-style command writes to

`ov version` writes to **stdout** (via `fmt.Println`) — the canonical
`layers/ov/layer.yml` test asserts a `stdout:` matcher.

```yaml
- id: ov-version
  command: /usr/local/bin/ov version
  exit_status: 0
  stdout:
    - matches: "[0-9]{4}\\.[0-9]+"
```

**But other tools genuinely write to stderr** — `ssh -V`, `openssl version`,
some `*-config --version` scripts. Always probe first
(`ov cmd <image> '<cmd> 2>&1 >/dev/null'` — if the output shows up here
it was stderr; if silent, it's stdout). See also the `ssh-version`
check in `layers/ssh-client/layer.yml` which IS a real stderr case.

### 6. Drop `$` anchor in `matches:` regexes on command output

Command stdout typically ends with `\n`. Go's `regexp` treats `$` as
end-of-input (not end-of-line by default), so `\.ipynb$` won't match
`path.ipynb\n`. Drop the anchor:

```yaml
# ❌ WRONG — fails because trailing newline
stdout:
  - matches: "\\.ipynb$"

# ✓ RIGHT — substring match is usually what you want
stdout:
  - matches: "\\.ipynb"
```

### 7. `${VAR:-default}` is NOT supported by ov's resolver

ov's variable resolver accepts `${IDENT}` only — no bash-style defaults,
no parameter expansion. An unresolved variable is reported as a
`unresolved variables: <name>` SKIP.

```yaml
# ❌ Parsed as a single var name; reported as unresolved if not set
command: psql -U ${POSTGRES_USER:-postgres}

# ✓ Plain env-var reference; resolves from the container's environment
# at deploy scope. For a true default, set it in the layer's `env:`.
command: psql -U ${POSTGRES_USER}
```

### 8. Fedora 43 package renames — query the installed name, not the dnf request

The `package:` verb does exact string equality against the rpm
database, not the dnf install request. When dnf resolves a virtual
provides, the test must name the actual installed package.

| dnf request | Installed name on Fedora 43 |
|-------------|-----------------------------|
| `redis` | `valkey-compat-redis` |
| `liberation-fonts` | `liberation-sans-fonts` (+ `-serif-fonts`, `-mono-fonts`) |

Always verify with `podman run --rm <image> rpm -qf /path/to/binary`.

### 9. Volume paths don't exist at build-scope

`/workspace`, `${VOLUME_CONTAINER_PATH:name}`, and similar paths
are provisioned by `ov config` at deploy time. A build-scope test on
them fails in `ov eval image` because the mount isn't attached.

```yaml
# ❌ Fails under `ov eval image` — workspace isn't mounted
- id: workspace-dir
  file: /workspace
  filetype: directory

# ✓ Scope to deploy so it runs only against live containers
- id: workspace-dir
  scope: deploy
  file: /workspace
  filetype: directory
```

### 10. Disposable tests run as the image's default USER — not root

`ov eval image` runs `podman run --rm <image>` with the image's USER
directive (typically uid 1000 for containers). Root-only files
(e.g. `/etc/sudoers.d/*` is root:root 0750) report as "missing" because
the user can't traverse the parent directory.

```yaml
# ❌ Reports "exists=false" even though the file IS there (0750 dir)
- id: sudoers-ov-user
  file: /etc/sudoers.d/ov-user
  exists: true

# ✓ Verify the semantic (can I sudo?) instead of the file bit
- id: sudoers-ov-user
  command: sudo -n -l
  exit_status: 0
  stdout:
    - contains: "NOPASSWD"
```

### 11. **Bootc images keep USER=root** — tests must cover both modes

Gotcha 10 flips for bootc. `ov/generate.go` deliberately omits the final `USER <uid>` directive on bootc images because systemd (PID 1 in a bootc VM) manages user sessions via login, so the container's own USER directive is irrelevant. Result: the same `ov eval image` that runs as uid 1000 in a container runs as **uid 0** in a bootc image.

That breaks any test that assumes the user-context default. A `sudo -n -l` check that expects `NOPASSWD` in the output works for non-bootc (USER=1000 → sudo lists the user's NOPASSWD rule) but fails for bootc (USER=0 → sudo prints root's Defaults block, which doesn't contain the literal string `NOPASSWD`). Same failure mode for any `test -f ~/.config/foo` that depends on `$HOME=/home/user`.

The fix: drop into `user` explicitly when running as root, stay as-is otherwise. Use **`runuser -u user -- <cmd>`** as the bridge — it's portable across util-linux versions and preserves stdout:

```yaml
# Dual-mode: container (USER=1000) AND bootc (USER=root) both pass
- id: sudoers-ov-user
  command: |
    if [ "$(id -u)" = "0" ]; then
      runuser -u user -- sudo -n -l
    else
      sudo -n -l
    fi
  exit_status: 0
  stdout:
    - contains: "NOPASSWD"
```

**Don't use `runuser -l user -s /bin/bash -c '…'`.** On Arch's util-linux
(2.42+), `runuser -l … -c` swallows the wrapped command's stdout when
the shell is a login shell — reproduced: `runuser -l user -s /bin/bash
-c 'sudo -n -l'` prints nothing and exits 0; `runuser -u user -- sudo
-n -l` prints the full NOPASSWD listing. This bit the earlier shipping
form of the test; the sshd layer now uses `-u … --`.

Worked example: `/ov-coder:sshd` ships exactly the `-u user --` pattern. Alternative: make the check `scope: deploy` so it runs against a live container where you control the user-switch externally.

### 12. **`ov eval image <short-name>` is ambiguous with multiple CalVer tags** — use the full registry ref

`ov eval image` resolves its positional argument against **local podman storage**, not `image.yml`. When the host has accumulated many CalVer tags for the same image (a normal consequence of iterative `ov image build` runs), the short form errors out:

```
ov: error: ambiguous short name "openclaw-desktop" in local storage;
           candidates: ghcr.io/overthinkos/openclaw-desktop:latest,
           ghcr.io/overthinkos/openclaw-desktop:2026.109.1418,
           ... Re-run with a full ref.
```

Use the fully-qualified registry ref:

```bash
ov eval image ghcr.io/overthinkos/openclaw-desktop:latest
```

This is different from `ov image inspect`, `ov image build`, and `ov eval live` (the live-service runner), which key off `image.yml` and accept short names unambiguously. Only the disposable-container runner has this restriction because it does not consult `image.yml` at all.

## Three levels, three sections

| Section | Authored in | When it runs |
|---------|-------------|--------------|
| `layer` | `eval:` in `layers/<name>/layer.yml` (scope:"build") | `ov eval image` + `ov eval live` |
| `image` | `eval:` in `image.yml` per image (scope:"build") | `ov eval image` + `ov eval live` |
| `deploy` | `eval:` with `scope: deploy`, or `deploy_eval:` in `image.yml`, or local `deploy.yml` `eval:` | `ov eval live <name>` only (deploy-scope checks need a running deployment with port mappings, volumes, and resolved runtime variables) |

The build label `org.overthinkos.eval` contains all three sections with
`origin:` annotations (`layer:<name>`, `image:<name>`, `deploy-default`,
`deploy-local`). `CollectEval` walks the base-image chain with a
visited-image guard: cycles are reported by `validateImageDAG` at validate
time, but the collector itself terminates cleanly even if called on a
pathological config.

## Verb routing (which executor runs each check)

Every verb dispatches through one of two executors depending on the run
mode — this is what makes the same `eval:` list work both against a
disposable container (`ov eval image`) and a running service (`ov eval live`):

| Verb + attributes | Under `ov eval live` (running service) | Under `ov eval image` (disposable) |
|-------------------|-----------------------------------|------------------------------------|
| file, package, service, process, user, group, interface, kernel-param, mount | `ContainerExecutor` (`podman exec`) | `ImageExecutor` (`podman run --rm`) |
| port with `listening` | `ContainerExecutor` (`ss`/`netstat` inside) | `ImageExecutor` |
| port with `reachable` | Host-side `net.DialTimeout` | Skipped (no host port binding on a disposable run) |
| command (`in_container: true`, default) | `ContainerExecutor` | `ImageExecutor` |
| command (`in_container: false`) | Host-side `exec.Command` | Skipped |
| http, dns, addr | Host-side (from the `ov` process) | In-container `curl` / `getent hosts` / `nc` |
| matching | In-process matcher eval | Same |

The routing table lives in `ov/testrun.go` (`runOne` switch) and
`ov/testrun_verbs.go`. When a check is unroutable (e.g. `port:
reachable` under `ov eval image`), the runner reports it as **skipped**
with a reason rather than failing the run.

### in-container `command:` stdin is guarded

An in-container `command:` script (`in_container: true`, the default) is
delivered to the pod shell over a stdin heredoc (`NestedExecutor.wrapWithJump`,
"stdin-attached exec"). The runner wraps every such script in
`{ <script>; } </dev/null` (`wrapContainerCommand`, `ov/evalrun.go`) so a
subcommand that reads stdin — `adb shell`, `ssh`, `read`, `cat` — cannot consume
the rest of the heredoc (the not-yet-run script lines, which would otherwise
truncate the check to its first command). Authors write plain multi-line
scripts; **no per-call-site `</dev/null` is needed**. Host-side `command:`
(`in_container: false`) runs via `sh -c` argv and is unaffected. For Android UI
readiness, prefer the typed `adb: wait-ui-settled` / `current-focus` / `keyevent`
verbs (`/ov-eval:adb`) over shell entirely.

## Runtime variables

`ov eval live` resolves these via `podman inspect` on the running container
before any check executes. `${NAME:arg}` is parameterized form.

| Variable | Source | Scope |
|----------|--------|-------|
| `${USER}`, `${HOME}`, `${UID}`, `${GID}` | Image metadata (OCI labels) | build + deploy |
| `${IMAGE}` | Image metadata | build + deploy |
| `${DNS}`, `${ACME_EMAIL}` | Image metadata + deploy.yml overlay | build + deploy |
| `${INSTANCE}` | `--instance` flag / deploy.yml key | deploy |
| `${CONTAINER_NAME}`, `${CONTAINER_IP}` | `podman inspect` | deploy |
| `${HOST_PORT:N}` | Port mapping for container port N | deploy |
| `${VOLUME_PATH:name}` | Host path backing the named volume (bind source, encrypted mount, or `_data` dir) | deploy |
| `${VOLUME_CONTAINER_PATH:name}` | In-container mount path for a volume | deploy |
| `${ENV_NAME}` | Effective env var value on the running container | deploy |

Build-scope checks may **not** reference deploy-scope variables — the
validator flags this at `ov image validate` time.

**No bash-style defaults**: `${VAR:-fallback}` is unsupported (see
Authoring Gotcha #7). Plain `${VAR}` only.

## Deploy.yml overlay rules

A local `deploy.yml` can contribute its own `eval:` list per image.
Merge rules applied by `ov eval live`:

1. Local entries with an `id:` matching a baked entry replace that entry.
2. Entries without a matching `id:` are appended.
3. To disable a baked check, reference it with `id:` and `skip: true`.

```yaml
# ~/.config/ov/deploy.yml
images:
  redis-ml:
    eval:
      - id: redis-responds                  # overrides image's baked check
        command: redis-cli -h 127.0.0.1 -p 16379 ping
        stdout: PONG
        in_container: false
      - id: external-tunnel                 # appended (no baked match)
        http: https://redis.tailnet.ts.net/health
        status: 200
      - id: old-probe                       # disable a legacy baked check
        skip: true
```

## Typical workflow

```bash
# 1. Author tests in layers/<name>/layer.yml.
# 2. Validate schema + references.
ov image validate

# 3. Build the image (tests are auto-embedded as LabelEval).
#    LABELs are emitted LAST in the final stage — a test edit rebuilds
#    in ~2 sec (cache hits every upstream RUN/COPY).
ov image build redis-ml

# 4. Run against a disposable container (build-scope checks only).
ov eval image redis-ml

# 5. Start the service and test end-to-end.
ov start redis-ml
ov eval live redis-ml

# 6. Override a baked deploy check via local deploy.yml, re-test.
$EDITOR ~/.config/ov/deploy.yml    # add tests: entry with matching id
ov eval live redis-ml
```

## Coverage snapshot (7 currently-tested images)

Reference numbers from the last end-to-end session:

| Image | Tests | Pass | Fail | Skipped |
|-------|-------|------|------|---------|
| `filebrowser` | 24 | 24 | 0 | 0 |
| `jupyter` | 32 | 32 | 0 | 0 (includes 3 declarative `mcp:` checks) |
| `openwebui` | 24 | 24 | 0 | 0 |
| `hermes` | 50 | 50 | 0 | 0 |
| `immich-ml` | 63 | 61 | 0 | 2 (redis port internal) |
| `selkies-desktop` | 91 | 91 | 0 | 0 |
| `sway-browser-vnc` | 92 | 92 | 0 | 0 (includes 7 declarative cdp/wl/dbus/vnc/mcp checks; 2 mcp tests come from the chrome-devtools-mcp layer) |

**Total: 376 checks across 7 images (0 failing).**

The "skipped" entries are intentional — they reference ports that
aren't mapped in the containing image's `port:` block. Skipping them
is correct behavior; they'd pass in an image that does expose those
ports.

## Realistic run-time expectations

`ov eval run default` is a multi-phase AI-iteration loop. Representative
per-phase durations (measured solving all 92/92 across 9 iterations on a
`disposable: true` eval-sandbox):

| Phase | Recipe | Typical iter1 duration | Notes |
|---|---|---|---|
| 1 | single-pod-system-state | ~10 min | trivial pod deploy |
| 2 | network-and-http | ~few min (cumulative scoring keeps phase 1 in scope) | nginx + curl in fedora:43 |
| 3 | cross-pod-nonce-traffic | ~few min | redis + redis-client; EVAL_NONCE_KEY/VALUE substituted per iter |
| 4 | mcp-protocol-probe | ~14 min | jupyter image build (~6 min) + 32 scenarios |
| 5 | kubernetes-cluster | ~20-30 min | k3s/kwok cluster bring-up |
| 6 | desktop-display-and-input | ~50 min iter1, ~10 min iter2 | sway-browser-vnc image build (~14 min) + 13 cdp/wl/vnc scenarios; commonly takes 2 iters due to Chrome SIGBUS instability |
| 7 | vm-control-libvirt-spice | ~40-50 min | libvirt domain (cloud-init, qemu-guest-agent) + 9 scenarios |
| 8 | nested-deployment-chain | ~40 min | socket-passthrough nested pods + 9 cross-hop scenarios |

**Total wall-clock for a fresh `ov eval run default`**: typically
**5-7 hours** on dev-class hardware. Set expectations accordingly — this
is NOT a quick smoke. For a fast canary that exercises the loader +
scoring paths end-to-end without real AI work, run
`ov eval run scaffolding-selftest` (~5 min, single recipe
`composition-import-selftest`).

The plateau bound is `score.<name>.plateau_iteration` (default 3). In
**progressive mode**, plateau ENDS the whole run — it does not advance
to the next phase, so an AI that stalls on one phase doesn't collect
easy wins on later phases it never engaged with.

## Issue 52328 (claude-code Bash + pgrep self-match deadlock) defense

The Claude Code runtime has a known issue documented in this project's
own `.eval/ISSUE-claude-code-bash-pgrep-self-match-deadlock.md`: when
the AI uses `Bash{run_in_background: true}` with a `until ! pgrep -f
"<pattern>"` poll-loop, the literal pattern string is preserved in the
spawned bash's argv (because the wrapper uses `bash -c '… eval
"<command>" …'`). `pgrep -f` then **matches itself**, the loop spins
forever, the parent `claude -p` process can't exit because it waits
for all background bash subprocesses, and the harness orchestrator
deadlocks waiting on claude.

The harness defends against this pattern between iterations: at every
iter end (in `eval_loop.go`'s `commitIterationBestEffort`), the
orchestrator runs
`podman exec ov-eval-sandbox pkill -f "while true.*sleep [0-9]+"` to
clean up orphaned keepalive bashes left dangling by the AI's
`TaskOutput`-timeout pattern, and logs the kill count.

Workaround for the AI inside the loop: prefer `kill -0 <PID>`-style
checks (track the original PID via `cmd & echo $!`) over `pgrep -f`
when the pattern would also match the wrapping bash. The harness
defense is a safety net, not a license to use the broken pattern
deliberately.

## Related skills

- **Live-container probe verbs under `ov eval`** — `/ov-eval:cdp`, `/ov-eval:wl`, `/ov-eval:dbus`, `/ov-eval:vnc`, `/ov-build:ov-mcp-cmd`, `/ov-eval:record`, `/ov-eval:spice`, `/ov-eval:libvirt`, `/ov-kubernetes:eval-k8s` are dispatched as `ov eval cdp|wl|dbus|vnc|mcp|record|spice|libvirt|k8s`. See the Subcommands section above.
- `/ov-image:layer` — layer authoring; `eval:` field is part of every `layer.yml`.
- `/ov-image:image` — image-level `eval:` and `deploy_eval:` at composition time.
- `/ov-core:deploy` — local `deploy.yml` overlay rules and the `eval:` merge.
- `/ov-build:validate` — static schema + cross-scope variable checks; the first
  gate before `ov image build`.
- `/ov-build:build` — how eval entries are embedded into the OCI label at build
  time; LABELs-at-end cache behavior.
- `/ov-build:inspect` — view the merged 3-section eval structure as JSON.
- `/ov-build:migrate` — `ov migrate` brings legacy configs (including
  any residual `harness.yml`) up to the current `eval.yml` schema.
- `/ov-internals:go` — implementation map: `evalspec.go`, `evalvars.go`,
  `evalrun.go`, `evalrun_verbs.go`, `evalrun_ov_verbs.go`,
  `evalcollect.go`, `eval_cmd.go`, `eval_runner_cmd.go`, `eval_loop.go`,
  `eval_runner_live.go`, `eval_watchdog.go`, `validate_eval.go`,
  `mcp.go`, `mcp_client.go`, plus the `LabelEval` constant in `labels.go`.
- `/ov-internals:generate-source` — how `LabelEval` is written into the Containerfile
  via `writeJSONLabel`, and why the LABEL block lives at the end of the
  final stage.
- `/ov-internals:capabilities` — the `LabelEval` three-section OCI label is
  part of the same capability contract as `LabelServices`.
- `/ov-internals:agents` — the sub-agents (`eval-bed-runner`,
  `deploy-verifier`) and dynamic workflows (`/verify-beds`,
  `/audit-deploy-configs`) that DRIVE these beds, and the R10/disposable/
  paste-proof rules that bind any agent or workflow running `ov eval run`.

## When to Use This Skill

**MUST be invoked** before authoring, running, or debugging eval
behavior at any level. Triggers: `ov eval` (any subcommand), `ov eval
run`, `ov eval image`, `ov eval live`, the `eval:` / `deploy_eval:`
YAML fields, the `org.overthinkos.eval` OCI label, `kind: ai` /
`kind: recipe` / `kind: score` / `kind: eval` in `eval.yml`,
`ov eval run <bed>` / `--all-beds` (kind:eval R10 beds), or any check
verb by name (file/port/http/command/package/service/cdp/wl/dbus/vnc/mcp/...).
