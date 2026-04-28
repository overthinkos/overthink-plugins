---
name: disposable
description: |
  `disposable: true` is the ONE and ONLY authorization for autonomous
  destroy + rebuild via `ov rebuild`. MUST be invoked before any task
  involving `ov rebuild`, live verification on rebuildable targets, or
  marking a VM / container deploy as safe-to-nuke. Explains why
  disposability is a DEPLOY property (not an image property), the
  separation between load-bearing `disposable:` and informational
  `lifecycle:`, why derivation is deliberately absent, and how the
  flag makes live verification fearless on shared hosts.
---

# disposable — explicit opt-in for autonomous destroy + rebuild

## Schema v3 note (in-progress cutover)

Schema v3 makes `DeploymentNode.Disposable` the **sole source of truth** for disposability. `VmSpec.Disposable` is retained during the Phase 1-5 transition for backward compatibility with the legacy `ov rebuild arch` / `ov vm destroy --disk` paths, but the canonical field on any deployment entry (e.g. `deployments.images.arch-vm.disposable: true`) is what the unified dispatcher reads.

A latent bug was fixed alongside the refactor: `MergeDeployConfigs` previously dropped the `Disposable` and `Lifecycle` fields during project ↔ per-machine overlay merge — meaning a disposable flag on the project `deploy.yml` would vanish after merge with a user's `~/.config/ov/deploy.yml`. That's now an explicit merge (project-set OR overlay-set → true), with the same "later wins" rule when both set it.

## Why this exists

Live-deploy verification (CLAUDE.md R1, R10) is mandatory — and it's
much easier to carry out aggressively when you can freely `destroy →
rebuild → retest` a target without asking the user for permission
every time. But autonomous destroy is only safe on resources whose
owner explicitly authorized it. The `disposable: true` flag is that
authorization. Nothing else is.

On a shared host, unrelated production services with live users may
run alongside ov-managed resources. The `disposable: true` flag must
therefore be:

- **Explicit.** Default false. No derivation from anything else. A
  human writes `disposable: true` on a specific deploy or nothing
  happens.
- **Per-deploy.** Set on the running deployment, not on the image.
  The same image can be deployed as disposable on one host and
  non-disposable on another.
- **Multi-instance aware.** Each deploy entry (vms.yml kind:vm, or
  deploy.yml entry under `deploys:`) carries its own flag — two
  instances of the same image can sit at different tiers with
  different disposability.

## Schema — two fields, clear roles

```yaml
# DEPLOY-shaped YAML (vms.yml kind:vm entity, or deploy.yml entry):

disposable: <bool>    # LOAD-BEARING. Default false. Explicit opt-in.
                      #   `true` authorizes `ov rebuild <name>` to
                      #   destroy + rebuild + restart unattended.
lifecycle: <tier>     # INFORMATIONAL ONLY. Free-form human tag.
                      #   scratch|dev|test|qa|staging|prod|custom.
                      #   Has ZERO effect on disposability.
```

**There is deliberately no derivation.** `lifecycle: dev` does NOT
make a deploy disposable. A human reader might assume it would, so
the anti-derivation invariant is enforced by `ov/classification.go`
`IsDisposableFields()` (a one-liner that explicitly ignores
lifecycle) and a unit test at `ov/classification_test.go`
`TestVmSpec_LifecycleAloneDoesNotAuthorize`. If you find yourself
tempted to add "if lifecycle in {scratch,dev,test} then
disposable=true": don't. That hidden logic is the entire failure
mode this design avoids.

## Where the fields live

- `kind: vm` entries in `vms.yml` — the VM template. Applies to
  every instance unless overridden.
- `deploys:` entries in `deploy.yml` — the container per-deploy
  counterpart. Each instance of a container image has its own
  entry, so per-instance classifications are natural.
- Per-instance VM overrides (deferred to a follow-up): will live at
  `~/.local/share/ov/vm/<domain-name>/instance.yml`. Until then,
  VMs inherit the vms.yml template classification for every
  instance.

## Visible from outside the source tree

For VMs with either `disposable: true` or a `lifecycle:` tag, the
libvirt domain XML carries:

```xml
<metadata>
  <ov:disposable xmlns:ov="https://overthinkos.org/ns/ov/1.0">true</ov:disposable>
  <ov:lifecycle xmlns:ov="https://overthinkos.org/ns/ov/1.0">dev</ov:lifecycle>
</metadata>
```

`virsh dumpxml <domain> | grep ov:` tells you the classification
without opening vms.yml.

For container deploys, the authoritative source is `deploy.yml` —
`ov status <name>` reflects it at runtime.

## `ov rebuild <name> [-i <instance>]`

The autonomous path. Resolves `<name>` as either a kind:vm entity
(vms.yml) or a deploys entry (deploy.yml). Refuses if effective
disposable is `false` or unset, with a pointed remediation message.
Otherwise: destroy → rebuild → restart in a single orchestrated
sequence.

```bash
# Allowed (disposable: true):
ov rebuild arch

# Refused (no disposable flag, or disposable: false):
$ ov rebuild production-api
ov rebuild: "production-api" is not marked `disposable: true` in
deploy.yml (current lifecycle: (unset)).
  `ov rebuild` only acts on explicitly disposable deploys —
  lifecycle tags alone do NOT authorize autonomous destroy.
  To opt in: edit deploy.yml and add `disposable: true` to the
  entry, or run: ov deploy add production-api <ref> --disposable
```

Flags:
- `--dry-run` — print the sequence without executing.
- `--rebuild-image` — also rebuild the underlying image (default:
  reuse).
- `-i <instance>` — target a named instance.

## Opting a deploy in

For containers, pass `--disposable` to `ov deploy add`:

```bash
ov deploy add my-test fedora-test --disposable --lifecycle test
```

This writes both fields to the deploy.yml entry (flags can also be
passed independently). Omitting `--disposable` means the entry
stays non-disposable — safe default.

For VMs, edit `vms.yml` directly on the `kind: vm` entity. The
`disposable: true` field lives alongside `ram:`, `cpus:`, etc. Any
synced host picks up the change on next `ov vm create` / `ov
rebuild`.

## Multi-instance examples

### Containers

```yaml
# deploy.yml — each entry has an independent classification.
deploys:
  fedora-coder:         # main instance (prod): NO disposable field → NOT disposable
    lifecycle: prod

  fedora-coder-dev:     # dev instance: lifecycle tag does NOT authorize rebuild
    lifecycle: dev      # ← human tag only; this is NOT disposable.

  fedora-coder-qa:      # explicit opt-in
    lifecycle: qa
    disposable: true

  fedora-coder-scratch: # minimal: no tag, explicit disposable
    disposable: true
```

Running `ov rebuild fedora-coder-dev` fails with the refusal
message — the `lifecycle: dev` tag is informational; the deploy
still needs an explicit `disposable: true` to be rebuildable.
`ov rebuild fedora-coder-qa` succeeds.

### VMs

Today, classification applies to all instances of a kind:vm entity.
If you need per-instance overrides (e.g., `ov vm create arch -i test`
with different classification than `-i prod`), that requires the
per-instance override file at `~/.local/share/ov/vm/ov-<name>-<instance>/instance.yml`
— planned follow-up.

## Extensibility

`lifecycle:` is free-form string — `scratch`, `dev`, `test`, `qa`,
`staging`, `prod` are the canonical values but any tag works
(`demo`, `perf`, `nightly` …). Tags are informational.

If a future feature wants to link a tier to behavior (e.g.
auto-expire deploys tagged `scratch` after 7 days), add a NEW
explicit field (`disposable.ttl`, `auto_cleanup: true`, etc.) —
never make `lifecycle:` behaviorally load-bearing. The
anti-derivation invariant is what keeps this classification safe
on shared hosts.

## Cross-references

- `CLAUDE.md` — R10 "Verify on a `disposable: true` target; prove
  it on a fresh rebuild", plus the "Disposable-Only Autonomy"
  section.
- `/ov:eval` — the 10 testing standards; disposable-only deployment
  is Standard 4, fresh-rebuild re-verification is Standard 10.
- `/ov-vms:vms` — kind:vm schema, including `disposable:` and
  `lifecycle:` fields.
- `/ov-vms:arch` — canonical worked example.
- `/ov:deploy` — `--disposable` / `--lifecycle` flags on
  `ov deploy add`.
- `/ov:rebuild` — the rebuild verb command reference (not yet
  authored — currently living in this skill).

## When to Use This Skill

**MUST be invoked** for any task that involves:
- `ov rebuild <name>` (determining which targets can be rebuilt
  unattended).
- Authoring / editing `disposable:` or `lifecycle:` fields in
  vms.yml or deploy.yml.
- Running live verification on a rebuildable target (CLAUDE.md R10).
- Adding a feature that checks disposability (must use
  `IsDisposable()` / `IsDisposableFields()`, never derive from
  lifecycle).

Invoke this skill BEFORE reading `ov/classification.go` or the
related YAML files.
