---
name: disposable
description: |
  `disposable: true` is the ONE and ONLY authorization for autonomous
  destroy + rebuild via `ov update`. MUST be invoked before any task
  involving `ov update`, live verification on rebuildable targets, or
  marking a VM / container deploy as safe-to-nuke. Explains why
  disposability is a DEPLOY property (not an image property), the
  separation between load-bearing `disposable:` and informational
  `lifecycle:`, why derivation is deliberately absent, and how the
  flag makes live verification fearless on shared hosts.
---

# disposable — explicit opt-in for autonomous destroy + rebuild

## Schema v4 note (in-progress cutover)

Schema v4 makes `DeploymentNode.Disposable` the **sole source of truth** for disposability. `VmSpec.Disposable` is retained during the Phase 1-5 transition for backward compatibility with the legacy `ov update arch` / `ov vm destroy --disk` paths, but the canonical field on any deployment entry (e.g. `deployments.images.eval-arch-vm.disposable: true`) is what the unified dispatcher reads.

A latent bug was fixed alongside the refactor: `MergeDeployConfigs` previously dropped the `Disposable` and `Lifecycle` fields during project ↔ per-machine overlay merge — meaning a disposable flag on the project `deploy.yml` would vanish after merge with a user's `~/.config/ov/deploy.yml`. That's now an explicit merge (project-set OR overlay-set → true), with the same "later wins" rule when both set it.

## Why this exists

Live-deploy verification (CLAUDE.md R1, R10, and Risk Driven Development) is
mandatory — and it's much easier to carry out aggressively when you can freely `destroy →
rebuild → retest` a target without asking the user for permission
every time. But autonomous destroy is only safe on resources whose
owner explicitly authorized it. The `disposable: true` flag is that
authorization. Nothing else is.

`disposable: true` is the **lifecycle boundary of the candybox** (CLAUDE.md
"Candyboxing"): Overthink secures the box as a whole and stocks it with the full
toolset, and this flag is what makes that fully-stocked box safe to tear down and
rebuild unattended. The candy inside can be generous *because* the wall — and its
authorized teardown — is explicit.

On a shared host, unrelated production services with live users may
run alongside ov-managed resources. The `disposable: true` flag must
therefore be:

- **Explicit.** Default false. No derivation from anything else. A
  human writes `disposable: true` on a specific deploy or nothing
  happens.
- **Per-deploy.** Set on the running deployment, not on the image.
  The same image can be deployed as disposable on one host and
  non-disposable on another.
- **Multi-instance aware.** Each deploy entry (vm.yml kind:vm, or
  deploy.yml entry under `deploy:`) carries its own flag — two
  instances of the same image can sit at different tiers with
  different disposability.

## Schema — four orthogonal fields, clear roles

```yaml
# DEPLOY-shaped YAML (deploy.yml entry):

disposable: <bool>    # LOAD-BEARING authorization. Default false.
                      #   `true` authorizes `ov update <name>` to
                      #   destroy + rebuild + restart unattended.
lifecycle: <tier>     # INFORMATIONAL ONLY. Free-form human tag.
                      #   scratch|dev|test|qa|staging|prod|custom.
                      #   Has ZERO effect on disposability.
ephemeral: <block>    # LOAD-BEARING operational mandate. Default absent.
                      #   Presence means "MUST be destroyed as soon as
                      #   it isn't needed anymore". Implies
                      #   `disposable: true` automatically (the one
                      #   documented exception to anti-derivation —
                      #   see "The ephemeral exception" below).
preemptible: <l|blk>  # LOAD-BEARING resource-arbitration. Default absent.
                      #   HOLDER side: occupies the exclusive host-resource
                      #   token(s) in `holds:`; MAY be gracefully stopped to
                      #   free them for a claimant, then MUST be restarted
                      #   (disk + definition preserved). The INVERSE of
                      #   disposable. ORTHOGONAL to the three above — no
                      #   derivation either way. The claimant side is the
                      #   sibling `requires_exclusive: [token...]` list.
                      #   See "The resource-arbitration axis" below.
```

**Anti-derivation invariant — with one named exception**:

`lifecycle: dev` does NOT make a deploy disposable. A human reader
might assume it would, so the anti-derivation invariant is enforced
by `ov/classification.go` and a unit test
`TestVmSpec_LifecycleAloneDoesNotAuthorize`. If you find yourself
tempted to add "if lifecycle in {scratch,dev,test} then
disposable=true": don't. That hidden logic is the entire failure
mode this design avoids.

### The ephemeral exception

`ephemeral: ...` DOES imply `disposable: true`. This is the **only**
field allowed to derive disposability — because it strengthens the
contract rather than weakening it. "Must be destroyed when not
needed" can only be honored if "may be destroyed" is also true.

Specifically:
- `LoadDeployConfig` auto-promotes `Disposable=true` when an entry
  carries `ephemeral: ...`.
- `DeploymentNode.IsDisposable()` returns `Disposable || IsEphemeral()`
  so every consumer (including `ov update`) treats ephemerals as
  authorized.
- Authoring `disposable: false` together with `ephemeral: ...` is
  contradictory; the loader rejects (or auto-promotes; bool fields
  cannot distinguish "explicit false" from "default false", so the
  load-time auto-promote is the canonical behavior).

`ephemeral:` itself is the operational counterpart to `disposable:`:
- `disposable: true` says "this resource MAY be destroyed
  autonomously by `ov update`" — a permission.
- `ephemeral: true` (or block form) says "this resource MUST be
  destroyed autonomously when no longer needed" — a requirement,
  enforced by the eval-runner / BDD step keywords / TTL transient
  timer registered in `ov deploy add`.

The implication arrow is one-way. Disposable resources are not
necessarily ephemeral; ephemeral resources are always disposable.

## The resource-arbitration axis — `preemptible` + `requires_exclusive`

A physical host resource can sometimes be held by only ONE deployment at a
time — the canonical case is a GPU passed through to a VM via VFIO (exactly one
VM can bind the card). `preemptible` (HOLDER side) + `requires_exclusive`
(CLAIMANT side) let the resource arbiter (`ov/preempt.go`) free such a resource
on demand and give it back afterward.

```yaml
# HOLDER — a long-running deploy that occupies an exclusive resource and may
# yield it. Authored as a token-list shorthand or a block:
preemptible: [nvidia-gpu]            # shorthand → holds, default stop/restore
preemptible:
  holds: [nvidia-gpu]                # REQUIRED, non-empty: operator-chosen token name(s)
  stop: shutdown                     # graceful shutdown (default & only) — frees a VFIO device
  restore: always                    # always (default) | on-success

# CLAIMANT — a deploy/eval bed that needs sole use of the resource while it runs:
requires_exclusive: [nvidia-gpu]
```

**How it works.** Before a claimant is brought up (`ov eval run <bed>`, or a
standalone `ov vm create` / `ov start`), the arbiter finds every RUNNING
preemptible holder whose `holds:` intersects the claimant's
`requires_exclusive:`, **gracefully stops** each (waiting until it actually
powers off so the resource is truly released), records a crash-safe **lease**,
and lets the claim proceed. When the claim is released — the eval bed tears down,
or the persistent claimant is stopped/destroyed — the arbiter **restarts** the
holders. A transient (eval) claim auto-releases via `defer`; a persistent claim
releases on the claimant's teardown command.

**The token is a name, not a mechanism.** `nvidia-gpu` is an operator-chosen
label for the physical resource, deliberately decoupled from HOW each side
reaches it (a VM via a PCI hostdev, a pod via `--device`/CDI). The arbiter does
pure set-intersection on tokens, so the same token unifies pod-vs-VM contention.

**Crash-safety (the restore guarantee).** A holder is NEVER left permanently
stopped. The lease ledger (`~/.local/share/ov/preemption/leases.yml`) is written
*before* any holder is stopped, and "restore" means "start every listed holder
that isn't running" — so a crash at any point is recoverable. `ov preempt status`
lists active leases and flags STRANDED ones (claimant gone); `ov preempt restore
[claimant]` reconciles them (also run automatically at the next acquire).

**`restore:` policy.** `always` (default) restarts the holder regardless of the
claim's outcome — the holder MUST survive, so it comes back even if the eval
failed. `on-success` leaves the holder stopped on a FAILED claim (for operator
inspection); recover it with `ov preempt restore`.

**Orthogonal to disposable/ephemeral — no derivation.** `preemptible` neither
implies nor is implied by any other axis. A deploy may legitimately be BOTH
preemptible (the arbiter stops it) AND disposable (R10 may rebuild it); a test
holder is often both. Stopping a holder is graceful + reversible (disk + state
preserved) — the OPPOSITE of `disposable`'s destroy authorization. Enforced by
`ov/classification.go` (`IsPreemptible()` is independent of `IsDisposable()`).

## Where the fields live

- `kind: vm` entries in `vm.yml` — the VM template. Applies to
  every instance unless overridden.
- `deploy:` entries in `deploy.yml` — the container per-deploy
  counterpart. Each instance of a container image has its own
  entry, so per-instance classifications are natural.
- Per-instance VM overrides (deferred to a follow-up): will live at
  `~/.local/share/ov/vm/<domain-name>/instance.yml`. Until then,
  VMs inherit the vm.yml template classification for every
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
without opening vm.yml.

For container deploys, the authoritative source is `deploy.yml` —
`ov status <name>` reflects it at runtime.

## `ov update <name> [-i <instance>]`

Resolves `<name>` as either a kind:vm entity (vm.yml) or a deploys entry
(deploy.yml). It **NEVER refuses** on disposability: an explicit
`ov update` rebuilds ANY target — for a non-disposable, non-ephemeral
target it prints a one-line transparency note
(`noteUpdateDisposability` in `ov/update_deploy_dispatch.go`) and
proceeds. Sequence: destroy → rebuild → restart, ending in the shared
`ov deploy add <node>` layer re-apply for every live substrate (so a
config change — a newly-added layer or nested pod — takes effect on the
rebuilt target). `disposable: true` stays load-bearing as the
authorization for the **AI's AUTONOMOUS** destroy + rebuild (CLAUDE.md
R10) and the eval-runner's unattended fresh rebuild — NOT as an
`ov update` capability check. A human (or the AI WITH explicit
authorization) may `ov update` a non-disposable target directly.

```bash
# Disposable: the AI may run this autonomously (no human in the loop):
ov update arch

# Non-disposable: the command still proceeds — with a transparency note.
# (The AI needs explicit human authorization to run it; a human running
#  it directly is always fine.)
$ ov update production-api
Note: "production-api" is not marked `disposable: true` (lifecycle: (unset));
rebuilding it anyway per your explicit `ov update`.
```

Flags (authoritative list: `ov update --help`):
- `--build` — rebuild the artifact first instead of reusing it (pod:
  rebuild the image; vm: rebuild the qcow2 disk; local: n/a). Without
  it, `ov update` redeploys the artifact already in local storage (it
  does NOT auto-pull).
- `--tag <calver>` — pin the image CalVer tag.
- `-i, --instance <name>` — target a named instance.
- `--seed` / `--no-seed` / `--force-seed` / `--data-from <image>` —
  bind-backed-volume data-sync control.

## Opting a deploy in

For containers, pass `--disposable` to `ov deploy add`:

```bash
ov deploy add my-test fedora-test --disposable --lifecycle test
```

This writes both fields to the deploy.yml entry (flags can also be
passed independently). Omitting `--disposable` means the entry
stays non-disposable — safe default.

For VMs, edit `vm.yml` directly on the `kind: vm` entity. The
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

`ov update` itself runs on ANY of these — a human can rebuild
`fedora-coder-dev` directly and it proceeds with the
`noteUpdateDisposability` transparency note (the `lifecycle: dev`
tag is informational, never an authorization). What the
`disposable: true` flag gates is AUTONOMOUS rebuild: the AI (and the
eval-runner) may unattended-rebuild only `fedora-coder-qa` and
`fedora-coder-scratch`; rebuilding `fedora-coder-dev` autonomously
requires explicit human authorization, because a lifecycle tag does
NOT authorize autonomous destroy.

### VMs

Today, classification applies to all instances of a kind:vm entity.
If you need per-instance overrides (e.g., `ov vm create arch -i test`
with different classification than `-i prod`), that requires the
per-instance override file at `~/.local/share/ov/vm/ov-<name>-<instance>/instance.yml`
— planned follow-up.

### Per-host device overlay (`instance.yml` `libvirt:`)

The same per-domain `~/.local/share/ov/vm/<domain>/instance.yml` also carries a
`libvirt:` block — a per-host device overlay merged onto the `VmSpec` at
`ov vm create` (`VmInstanceOverride.ApplyToVmSpec`, before `RenderDomainXML`).
Only the HOST-SPECIFIC device categories merge — `devices.hostdevs` (a PCI
`<hostdev>` whose bus/slot address is host-specific) and `devices.filesystems`
(a virtiofs share rooted at an absolute host path) — appended to whatever the
committed `vm.yml` declares. This is where a GPU passthrough address and an
operator-home share live, OUTSIDE version control, so the project's `kind: vm`
entities stay PORTABLE: no PCI address, no operator-home path committed (a
card-less host simply omits the overlay, and the GPU-gated checks report N/A).
The overlay reuses the `kind: vm` `libvirt:` schema verbatim — the block is
identical to what `vm.yml` would carry; generate a hostdev block with
`ov vm gpu list`. The file still also carries `disposable:` / `lifecycle:`
(above), and yaml.v3 unknown-key tolerance keeps the formats independent. See
`/ov-internals:vm-spec` (`VmSpec.Libvirt`) and `/ov-vm:vm` (GPU passthrough).

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
  it on a fresh rebuild", plus the "Candyboxing", "Disposable-Only
  Autonomy", and "Risk Driven Development (RDD)" sections. `disposable: true`
  is the lifecycle boundary of the candybox — the wall that makes a
  fully-stocked, secured box safe to destroy and rebuild — and the live surface
  you prove a high-risk assumption on BEFORE editing (RDD), never trusting a doc
  or the code for a high-risk call.
- `/ov-eval:eval` — the 10 testing standards; disposable-only deployment
  is Standard 4, fresh-rebuild re-verification is Standard 10.
- `/ov-vm:vms-catalog` — kind:vm schema, including `disposable:` and
  `lifecycle:` fields.
- `/ov-vm:arch` — canonical worked example.
- `/ov-core:deploy` — `--disposable` / `--lifecycle` flags on
  `ov deploy add`.
- `/ov:rebuild` — the rebuild verb command reference (not yet
  authored — currently living in this skill).

## When to Use This Skill

**MUST be invoked** for any task that involves:
- `ov update <name>` (determining which targets can be rebuilt
  unattended).
- Authoring / editing `disposable:` or `lifecycle:` fields in
  vm.yml or deploy.yml.
- Running live verification on a rebuildable target (CLAUDE.md R10).
- Adding a feature that checks disposability (must use
  `IsDisposable()` / `IsDisposableFields()`, never derive from
  lifecycle).

Invoke this skill BEFORE reading `ov/classification.go` or the
related YAML files.
