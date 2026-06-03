---
name: ov-update
description: |
  Update image and restart service with data sync.
  MUST be invoked before any work involving: ov update command, pulling new image versions, data seeding, force-seed, or updating deployed services.
  Named `ov-update` (not `update`) to disambiguate from Claude Code's built-in `/update`/`/upgrade` slash commands.
---

# ov update -- Update Image and Restart

## Overview

Redeploy the current artifact and restart the service — for EVERY deploy kind through ONE codepath. `ov update <name>` resolves the deploy via `ResolveTarget` and calls `LifecycleTarget.Rebuild` (`ov/unified_targets_*.go`); there is no per-kind update code. The unified contract is **redeploy the current artifact + restart by default; `--build` rebuilds the artifact first** — realized per substrate: pod → `deploy add → config → start` (`--build` rebuilds the image); vm → destroy→create the domain (reuse the qcow2 disk unless `--build`); local → re-apply layers idempotently. k8s has no live runtime to rebuild (apply it via `kubectl apply -k`), so `ov update <k8s>` errors uniformly. Data sync uses MERGE mode by default — adds new files without overwriting existing user modifications.

**`ov update` does NOT auto-pull.** It redeploys the image already in local storage. To advance a deploy to a newer published image, `ov image pull <ref>` first, then `ov update`; or `ov update --build` to rebuild locally. (This consistency with vm's reuse-disk default replaced the former pod-only auto-pull, so `ov update` behaves identically across kinds.) See `/ov-core:deploy` for the unified command family and `/ov-local:local-deploy` for host-target specifics.

**`ov update` obeys an EXPLICIT invocation on ANY target.** It does NOT refuse a non-`disposable: true` deploy — for a target that is neither disposable nor ephemeral it prints a one-line transparency note (`noteUpdateDisposability`, naming the deploy key + lifecycle, so the operator can catch a mistyped name) and proceeds with the rebuild. The `disposable:` flag stays load-bearing as the authorization for the AI's AUTONOMOUS destroy + rebuild (CLAUDE.md R10) and for the eval-runner's unattended fresh-rebuild (`validateEvalBeds`); it does NOT gate this human-driven verb. See `/ov-internals:disposable`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Update | `ov update <image>` | Pull new image, seed data, restart |
| Skip data sync | `ov update <image> --no-seed` | Pull and restart without data sync |
| Force overwrite | `ov update <image> --force-seed` | Overwrite existing data (cp -a) |
| Build instead of pull | `ov update <image> --build` | Build image locally before restart |
| Specific tag | `ov update <image> --tag v2.0` | Update to a specific tag |
| Named instance | `ov update <image> -i INSTANCE` | Update a named instance |
| Cross-image data | `ov update <image> --data-from <other>` | Seed data from a different image |

## Data Sync Modes

| Flag | Mode | Behavior |
|------|------|----------|
| `--seed` (default) | MERGE | `cp -an` -- adds new files, preserves existing |
| `--no-seed` | SKIP | No data sync, just pull and restart |
| `--force-seed` | OVERWRITE | `cp -a` -- overwrites all existing data |

## Usage

### Standard Update

```bash
# Pull latest image, merge new data files, restart
ov update jupyter
```

### Build and Update

```bash
# Build locally then update
ov update jupyter --build
```

### Force Data Reset

```bash
# Overwrite all data with fresh image defaults
ov update jupyter --force-seed
```

### Cross-Image Data Source

```bash
# Use data layers from a different image
ov update jupyter --data-from jupyter-custom
```

### Per-Instance Update with Volume Preservation

```bash
ov update selkies-desktop -i 82.23.94.69
```

When using `-i INSTANCE`, the update operates on a single named instance. The
container is destroyed and recreated from the new image, but the per-instance named
volumes are reattached:

- `ov-selkies-desktop-82.23.94.69-chrome-data` → `/home/user/.chrome-debug`
- `ov-selkies-desktop-82.23.94.69-selkies-config` → `/home/user/.config/selkies`

User-side state in those volumes (Chrome cookies, profile, history, selkies client
settings) **survives the restart**. The cgroup is recreated fresh, so any in-memory
state — including any leaked memfd-backed shmem from the old container — is released.

To roll a fleet of instances forward, loop the per-instance update:
`for ip in ...; do ov update selkies-desktop -i $ip; done` — each active
streaming session resumes cleanly on the new image with its per-instance
volume state intact.

### Rollback via `podman tag`

`ov update` does not have a built-in rollback flag, but the previous CalVer-tagged image
is left in the local podman image store after each `ov image build`. To roll back:

```bash
# Find the previous tag
podman image ls ghcr.io/overthinkos/selkies-desktop
# REPOSITORY                            TAG             IMAGE ID
# ghcr.io/overthinkos/selkies-desktop   latest          bc2bb4f90ca0   <- new (broken)
# ghcr.io/overthinkos/selkies-desktop   2026.102.2333   bc2bb4f90ca0
# ghcr.io/overthinkos/selkies-desktop   2026.102.1933   502c8012c7a5   <- previous

# Re-point :latest at the previous tag
podman tag ghcr.io/overthinkos/selkies-desktop:2026.102.1933 \
           ghcr.io/overthinkos/selkies-desktop:latest

# Restart the service(s) — they pick up :latest, no re-pull needed
systemctl --user restart ov-selkies-desktop.service
# or per-instance:
systemctl --user restart ov-selkies-desktop-82.23.94.69.service
```

This is fast (no network round-trip) and survives because podman's image GC is
opt-in. To make rollback survive a `podman image prune`, also tag the previous image
with a stable name (e.g., `:rollback`). The deploy.yml entry is unchanged — only the
local registry pointer moves.

## Behavior by Mode

### Quadlet Mode

1. Pull/build new image
2. Sync data from data layers into the image's volumes — both bind mounts and podman named volumes (if `--seed`)
3. `systemctl --user restart ov-<image>.service`
4. Update `deploy.yml` with new `data_source`

### Direct Mode

1. Pull/build new image
2. Sync data from data layers into the image's volumes (if `--seed`)
3. Print restart instructions (manual restart required)

## Data Seeding

Data layers (layers that declare a `data:` block in `layer.yml`) ship
starter content that gets copied into the runtime volume on first
deployment. Examples: `notebook-templates` ships
`getting-started.ipynb` into jupyter's `workspace` volume;
`notebook-finetuning` ships a set of Unsloth notebooks into
`jupyter-ml-notebook`.

### Which volumes get seeded

Both backings are seeded:

- **Bind-mounted volumes** (`type: bind` in `deploy.yml`) — the
  staged data is copied into the host directory via a throwaway
  `podman run` with `--userns=keep-id` so the files end up owned by
  the real host user.
- **Named volumes** (the default when no `type: bind` override is set)
  — the staged data is copied via `podman run -v <name>:/seed`
  *without* `--userns=keep-id`, so the files match the rootless
  subuid identity the runtime container uses.

### Mode semantics

- **`DataProvisionInitial`** (the default for `ov config`) — only
  seeds when the target volume is empty. For bind mounts, checks the
  per-entry subdirectory; for named volumes, checks the volume root
  via `podman volume inspect`.
- **`DataProvisionMerge`** (the default for `ov update --seed`) —
  always runs `cp -an`, which adds new files without overwriting
  existing ones. Safe on non-empty targets.
- **`DataProvisionForce`** (`ov update --force-seed` or
  `ov config --force-seed`) — runs `cp -a` unconditionally,
  overwriting existing files.

When seeding populates a named volume, the output shows
`  <volume> (named): provisioning from /data/<volume>/ ...`.

## Cross-References

- `/ov-build:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov-core:ov-config` -- initial deployment setup
- `/ov-core:start` -- start a service
- `/ov-build:build` -- building images locally
- `/ov-core:ov-status` -- check service status after update

## Live-deploy verification is mandatory (see `/ov-eval:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-internals:disposable`). Use `ov update <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vm.yml.

**After committing the source-level fix, `ov update` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
