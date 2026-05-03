---
name: cutover-policy
description: |
  Authoritative reference for the "Hard Cutover by Default" policy governing
  schema changes, API renames, and deprecations. Forbidden patterns, required
  deliverables, rationale, examples from this repo, and the no-exception
  enforcement: plans are authored as full-scope single-phase cutovers and
  executed end-to-end regardless of estimated time, context, or scope.
  MUST be invoked when planning or reviewing any breaking change to Go types,
  YAML field names, CLI flags, or OCI labels.
---

# cutover-policy

Every schema change, API rename, or deprecation in Overthink ships as a **single hard-cutover PR**. This applies to BOTH code (Go types, exported functions, CLI flags, OCI labels) AND config (overthink.yml, deploy.yml, layer.yml, vms.yml field names and shapes).

A cutover may NEVER be phased — not at plan authoring, not at execution. There is no pre-approval split, no post-approval split, no phased rollout, no grace period, no "author it as two plans" fallback. Plans are authored as full-scope, single-phase cutovers regardless of estimated time, scope, or context. Every cutover executes end-to-end through R10 in the SAME conversation. ALWAYS push as far as you can; compact context and continue, as many times as it takes. An approved plan is a CONTRACT; implement it as written.

This skill is the source of truth for the policy. `CLAUDE.md` links here rather than re-stating the full policy inline. The project's `UserPromptSubmit` hook at `.claude/hooks/runtime-verification-reminder.sh` mirrors the key directives and fires at every user prompt.

## One phase, many tasks, one cutover — the workflow

**A "phase" is the whole cutover. "Tasks" are the breakdown inside it.** Never treat tasks as separate phases with their own sign-off.

1. **Plan**: write a plan file that describes the cutover as ONE phase. Decompose into tasks with `TaskCreate`. The plan file names the cutover, not a sequence of cutovers.
2. **Implement**: execute every task in the same working tree. Transitional aliases, legacy-accepting code paths, or temporary dual-dispatch are permitted DURING implementation. They MUST be deleted before the end of the same cutover.
3. **Test at the end, not between tasks**: run unit tests, `ov image build`, `ov deploy add` + `ov test`, and the R10 fresh-rebuild re-verification AFTER all tasks are marked complete. Testing between tasks is cheap smoke-confirmation; the acceptance gate is the full-stack run against the final code.
4. **Ship or fix**: if any verification step fails, fix it in the same working tree and re-run the full verification. Do NOT commit a partial state.

**Forbidden**: "Phase 1 landed, Phase 2 pending" as a stopping point. That leaves the system half-migrated — legacy paths live alongside new paths, migrations not yet run, tests passing for some beds and not others. Every historical instance of that pattern in this project left dead code and untested integration points that bit users later.

**Splitting a cutover across conversation turns is ABSOLUTELY FORBIDDEN, with NO exception — at plan authoring or at execution.** Once a plan is approved, it executes end-to-end through R10 in the same conversation. ALWAYS push as far as you can. Compact context and continue, as many times as it takes. Time, context space, session budget, scope size, and "the work turned out to be large" are NEVER valid stop reasons.

Do not propose phasing, narrowing, or scope reduction at plan-authoring time. Do not negotiate a split mid-execution. Do not silently downgrade. An approved plan is a CONTRACT; implement it as written. The ONLY valid stop conditions, at any stage, are (a) an error you cannot resolve that requires user input, or (b) the plan contradicts itself, CLAUDE.md, or a loaded skill — STOP and ask in either case; do NOT commit a partial state.

## Forbidden patterns (by default)

- **Backcompat unmarshalers** that accept both old and new YAML forms.
- **`deprecated.go` shims** or type aliases that re-export removed identifiers.
- **Silent upconverters** that rewrite stale configs at load time.
- **Dual-mode code paths** where both the old and new surface work simultaneously.
- **"Phase 2 cleanup" comments** or TODOs for work that the cutover PR was supposed to complete.

Each of these has a specific failure mode that has occurred historically: the first three drag legacy-state complexity forward indefinitely; the fourth multiplies the test matrix; the fifth is the anti-pattern `R6` in `CLAUDE.md` forbids on a per-plan basis.

## Required for every breaking change

- A one-shot **`ov migrate <name>`** command that transforms legacy configs in-place. Migration commands are **idempotent** — running twice is a no-op. See `/ov-build:migrate`.
- **Hard load-time errors** for any residual legacy field, with a one-line remediation hint pointing at the migration command.
- **Deletion — in the same PR** — of every Go type, function, CLI flag, OCI label, YAML field, skill doc paragraph, and test fixture that references the removed surface.

## Rationale

Phased migrations accumulate mid-state complexity that, in practice, rarely gets removed. "We'll clean up in Phase 2" is a fiction that the history of this project has shown over and over. Making hard cutover the default across the project closes the loophole where this behavior sneaks in via PRs whose plans didn't explicitly call for a clean cutover.

## No exception clause

There is no pre-approval split, no post-approval split, no phased rollout, no grace period, no "resume in the next session", no "author it as two plans" fallback. Plans are authored as full-scope, single-phase cutovers regardless of estimated time, scope, or context. Phase / scope / time concessions are FORBIDDEN at plan authoring AND at execution.

Every cutover — regardless of estimated effort — runs as ONE phase in the SAME conversation through R10. ALWAYS push as far as you can. Compact context and continue, as many times as it takes.

The ONLY valid stop conditions, at any stage, are:

1. An error you cannot resolve that requires user input.
2. The plan contradicts itself, CLAUDE.md, or a loaded skill.

In either case STOP and ask. Do NOT silently downgrade scope, narrow tests, abbreviate the R10 matrix, or commit a partial state. An approved plan is a CONTRACT; implement it as written.

Forbidden internal-voice triggers (each is a confession, NOT a defence):

- "this is too large for one turn"
- "we should split this into two plans"
- "session budget concerns mean ..."
- "to fit context space ..."
- "for tractable wall-clock ..."
- "I'll narrow scope just for this canary ..."
- "let me ask whether to phase it"

If you catch yourself forming any of those: STOP forming them. The plan executes as written.

## Worked example: the VM refactor (this cutover)

Good reference implementation. Deliverables in one PR:

### Code deletions

- `VmConfig` struct — deleted from `ov/config.go`.
- `ImageConfig.Vm`, `ImageConfig.Libvirt` fields — deleted.
- `ResolvedImage.Vm` — deleted.
- `resolveVmConfig` function — deleted.
- `LabelVm`, `LabelLibvirt` label constants — deleted from `ov/labels.go`.
- `CapabilityLabelMap` entries for `Vm` and `Libvirt` — deleted.
- Image-level libvirt snippet validation in `ov/validate.go` — deleted.
- Image-level libvirt iteration in `ov/libvirt.go` — deleted.

### Schema deletions

- `image.bootc: true` + `image.vm: {...}` + `image.libvirt: [...]` — all three rejected by the loader with hard errors.

### Replacement surface

- `vms.yml` with `kind: vm` entities.
- `VmSpec` + `VmSource` + `LibvirtConfig` + `VmCloudInit` + friends in `ov/vm_spec.go`, `ov/cloud_init_types.go`, `ov/libvirt_schema.go`.
- `vm:<name>` deploy target via `VmDeployTarget`.

### Migration

- **`ov migrate vm-spec`** in `ov/migrate_vm_spec.go`. Idempotent. Harvests legacy fields into `vms:` entries, preserves pre-existing `vms:` keys, never clobbers user customizations.

### Load-time error

Old projects loading under new code get:

```
Error: image entry "foo" declares legacy field "bootc: true".
Run: ov migrate vm-spec
```

Remediation hint points at the migration command directly.

### Documentation refresh

All referring skills revised in the same sweep (this documentation-update PR). No stale references to `VmConfig`/`LabelVm`/`LabelLibvirt`/`image.bootc`/`image.vm:`/`image.libvirt:` anywhere in `plugins/`, `README.md`, or `CLAUDE.md`.

### Test deletions

`labels_test.go`, `libvirt_test.go`, `config_test.go` lost the fixtures and assertions that exercised the legacy surface. New fixtures exercise VmSpec instead.

## What a cutover PR looks like

The commit graph for this VM cutover:

```
089f375 (initial D1-D17 tasks — new VmSpec surface lands alongside legacy)
 ↓
b249ee4 (arch live-tested + migrate vm-spec authored)
 ↓
3087e0a feat(ov)!: hard cutover — delete VmConfig, ImageConfig.Vm/Libvirt, OCI labels
```

The `!` in the last commit marker is the Conventional Commits breaking-change indicator. The commit body lists every deleted identifier, every deleted YAML field, every updated test. `ov migrate vm-spec` is runnable against old projects from the point of this commit forward, no additional steps.

## Historical cutovers that followed this policy

- Unified YAML cutover (legacy `image.yml`/`build.yml`/flat-form `layer.yml` → `overthink.yml` with kind-keyed wrappers + `includes:` + `discover:`) — `ov migrate unified`.
- Unified `service:` schema cutover (legacy `service: |...|` raw INI and `system_services:` → structured `service:` list with 22 fields incl. `kind: eventlistener`) — folded into `ov migrate unified`.
- User policy cutover (rename-based user renaming → `base_user:` + `user_policy:` declarative matrix) — no separate migration; hard cutover delete + skill updates.

Each followed the same three-step shape: **delete old surface + publish migration + hard load error**. See `/ov-build:migrate` for the command surface.

## When the policy might not apply

- **Purely additive changes** — new field that defaults to "off" with no removal of existing surface. No cutover needed; just add it.
- **Internal refactoring** — a Go function rename that isn't part of any stable API. No cutover needed; normal refactor rules apply.
- **Bug fixes** — behavior change without schema or API change. No cutover.

The policy kicks in when the change is **visible to consumers** (YAML authors, other Go packages, OCI-label readers) AND removes something that was previously usable.

## Cross-References

- `/ov-build:migrate` — command-surface reference; lists all `ov migrate <name>` sub-verbs
- `/ov-dev:vm-spec` — example output of a cutover (new types replacing deleted ones)
- `/ov-dev:capabilities` — example of coordinated label-map cleanup during a cutover
- `/ov-dev:install-plan` — shared IR that survived the cutover unchanged (non-example — additive extension of the DeployTarget surface)
- CLAUDE.md "Hard Cutover by Default" — summary pointing at this skill

## Live-deploy verification is mandatory (see `/ov-build:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-dev:disposable`). Use `ov rebuild <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vms.yml.

**After committing the source-level fix, `ov rebuild` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
