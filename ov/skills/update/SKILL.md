---
name: update
description: |
  Update image and restart service with data sync.
  MUST be invoked before any work involving: ov update command, pulling new image versions, data seeding, force-seed, or updating deployed services.
---

# ov update -- Update Image and Restart

## Overview

Pull or build a new image version, optionally sync data from data layers into the image's volumes (bind mounts AND podman named volumes — both kinds are seeded), then restart the service. Data sync uses MERGE mode by default -- adds new files without overwriting existing user modifications.

**Relationship to `ov deploy add --pull`** — `ov update <name>` is the container-deploy equivalent of `ov deploy add <name> --pull`. For host deploys, the same semantics are reached via `ov deploy add host --pull`. See `/ov:deploy` for the unified command family and `/ov:host-deploy` for host-target specifics.

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

This was the rollout pattern used in commit `7977b91` (pixelflux dmabuf cache leak
fix): all 13 selkies-desktop instances were updated via a `for ip in ...; do ov update
selkies-desktop -i $ip; done` loop, and every active streaming session resumed cleanly
on the new image with state intact.

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

Before the fix in `fix/data-seeding-complete`, seeding ran only for
bind mounts — named volumes were silently skipped. Hosts running that
pre-fix code will see their previously-empty named volumes populated
with starter content on the first `ov config` or `ov update` after
upgrading. Existing user-modified content is preserved in all cases.

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

### Upgrade note

The first `ov config` or `ov update` after upgrading to the fix will
seed data into named volumes that previously had none — you'll see
`  <volume> (named): provisioning from /data/<volume>/ ...` in the
output. This is the intended behavior; operator action is only
needed if the seeding reports an error.

## Cross-References

- `/ov:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov:config` -- initial deployment setup
- `/ov:start` -- start a service
- `/ov:build` -- building images locally
- `/ov:status` -- check service status after update

## Live-deploy verification is mandatory (see `/ov:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-dev:disposable`). Use `ov rebuild <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vms.yml.

**After committing the source-level fix, `ov rebuild` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
