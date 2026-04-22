---
name: cutover-policy
description: |
  Authoritative reference for the "Hard Cutover by Default" policy governing
  schema changes, API renames, and deprecations. Forbidden patterns, required
  deliverables, rationale, examples from this repo, and the exception clause.
  MUST be invoked when planning or reviewing any breaking change to Go types,
  YAML field names, CLI flags, or OCI labels.
---

# cutover-policy

Every schema change, API rename, or deprecation in Overthink ships as a **single hard-cutover PR** unless the user explicitly requests a phased migration. This applies to BOTH code (Go types, exported functions, CLI flags, OCI labels) AND config (overthink.yml, deploy.yml, layer.yml, vms.yml field names and shapes).

This skill is the source of truth for the policy. `CLAUDE.md` links here rather than re-stating the full policy inline.

## Forbidden patterns (by default)

- **Backcompat unmarshalers** that accept both old and new YAML forms.
- **`deprecated.go` shims** or type aliases that re-export removed identifiers.
- **Silent upconverters** that rewrite stale configs at load time.
- **Dual-mode code paths** where both the old and new surface work simultaneously.
- **"Phase 2 cleanup" comments** or TODOs for work that the cutover PR was supposed to complete.

Each of these has a specific failure mode that has occurred historically: the first three drag legacy-state complexity forward indefinitely; the fourth multiplies the test matrix; the fifth is the anti-pattern `R6` in `CLAUDE.md` forbids on a per-plan basis.

## Required for every breaking change

- A one-shot **`ov migrate <name>`** command that transforms legacy configs in-place. Migration commands are **idempotent** — running twice is a no-op. See `/ov:migrate`.
- **Hard load-time errors** for any residual legacy field, with a one-line remediation hint pointing at the migration command.
- **Deletion — in the same PR** — of every Go type, function, CLI flag, OCI label, YAML field, skill doc paragraph, and test fixture that references the removed surface.

## Rationale

Phased migrations accumulate mid-state complexity that, in practice, rarely gets removed. "We'll clean up in Phase 2" is a fiction that the history of this project has shown over and over. Making hard cutover the default across the project closes the loophole where this behavior sneaks in via PRs whose plans didn't explicitly call for a clean cutover.

A developer who *needs* a grace period should record that explicitly in the plan file — both to force an honest review of whether the grace period is actually necessary, and to preserve the audit trail when someone later asks why the old field lingered.

## Exception clause

Explicit user instruction in the form of:

- "keep the old API for a grace period"
- "phase the cutover across two releases"
- "ship backcompat for external consumers of the OCI labels"

The exception **must be recorded in the plan file**. When the plan is silent, hard cutover is the default.

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
b249ee4 (arch-cloud-base live-tested + migrate vm-spec authored)
 ↓
3087e0a feat(ov)!: hard cutover — delete VmConfig, ImageConfig.Vm/Libvirt, OCI labels
```

The `!` in the last commit marker is the Conventional Commits breaking-change indicator. The commit body lists every deleted identifier, every deleted YAML field, every updated test. `ov migrate vm-spec` is runnable against old projects from the point of this commit forward, no additional steps.

## Historical cutovers that followed this policy

- Unified YAML cutover (legacy `image.yml`/`build.yml`/flat-form `layer.yml` → `overthink.yml` with kind-keyed wrappers + `includes:` + `discover:`) — `ov migrate unified`.
- Unified `service:` schema cutover (legacy `service: |...|` raw INI and `system_services:` → structured `service:` list with 22 fields incl. `kind: eventlistener`) — folded into `ov migrate unified`.
- User policy cutover (rename-based user renaming → `base_user:` + `user_policy:` declarative matrix) — no separate migration; hard cutover delete + skill updates.

Each followed the same three-step shape: **delete old surface + publish migration + hard load error**. See `/ov:migrate` for the command surface.

## When the policy might not apply

- **Purely additive changes** — new field that defaults to "off" with no removal of existing surface. No cutover needed; just add it.
- **Internal refactoring** — a Go function rename that isn't part of any stable API. No cutover needed; normal refactor rules apply.
- **Bug fixes** — behavior change without schema or API change. No cutover.

The policy kicks in when the change is **visible to consumers** (YAML authors, other Go packages, OCI-label readers) AND removes something that was previously usable.

## Cross-References

- `/ov:migrate` — command-surface reference; lists all `ov migrate <name>` sub-verbs
- `/ov-dev:vm-spec` — example output of a cutover (new types replacing deleted ones)
- `/ov-dev:capabilities` — example of coordinated label-map cleanup during a cutover
- `/ov-dev:install-plan` — shared IR that survived the cutover unchanged (non-example — additive extension of the DeployTarget surface)
- CLAUDE.md "Hard Cutover by Default" — summary pointing at this skill
