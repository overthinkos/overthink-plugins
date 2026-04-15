---
name: enc
description: |
  MUST be invoked before any work involving: encrypted storage, ov config mount/unmount/status/passwd commands, gocryptfs, or encrypted volume backing.
---

# Enc - Encrypted Storage

## Overview

Encrypted volume backing is configured at deploy time via `ov config --encrypt <volume>`. Gocryptfs-encrypted volumes store sensitive data (credentials, keys, configs) with transparent encryption at rest. The cipher directory lives on disk; the plain directory is mounted on demand. `ov config <image>` handles initialization and mounting during deployment setup. `ov start` mounts encrypted volumes inline before starting the container.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Configure encrypted | `ov config <image> --encrypt <vol>` | Set volume backing to encrypted |
| Setup (init + mount) | `ov config <image>` | Initialize cipher dirs and mount encrypted volumes |
| Mount | `ov config mount <image>` | Mount encrypted volumes |
| Unmount | `ov config unmount <image>` | Unmount encrypted volumes |
| Status | `ov config status <image>` | Show mount status |
| Change password | `ov config passwd <image>` | Change encryption password |

All commands accept `--volume NAME` to target a specific volume (otherwise all encrypted volumes are affected).

## Configuration

Encrypted volumes are configured at deploy time, not build time. Use `ov config --encrypt` to set a layer-declared volume's backing to encrypted:

```bash
# Configure "library" volume as encrypted
ov config immich --encrypt library

# Or via canonical syntax
ov config immich -v library:encrypted

# Or via env var
OV_VOLUMES_IMMICH="library:encrypted" ov config immich --password auto
```

This saves to deploy.yml:
```yaml
volumes:
  - name: library
    type: encrypted
```

Rules:
- Volume must be declared in `layer.yml` (or provided as deploy-only with `path:`)
- `name`: matches a layer volume name, must match `^[a-z0-9]+(-[a-z0-9]+)*$`

### Per-Volume Explicit Path

Each encrypted volume can specify its own storage directory:

```bash
# Explicit per-volume paths (each volume gets its own directory)
ov config immich-ml \
  --volume library:encrypt:/mnt/nas/immich/library \
  --volume pgdata:encrypt:/mnt/nas/immich/pgdata

# Canonical syntax
ov config immich-ml -v library:encrypted:/mnt/nas/immich/library
```

The path is the **direct volume directory** — `cipher/` and `plain/` are created inside it:
```
/mnt/nas/immich/library/
  cipher/    # gocryptfs encrypted data
  plain/     # FUSE mount point
```

Without an explicit path, the global `encrypted_storage_path` is used with an `ov-<image>-<name>` prefix (backward compatible).

## Storage Layout

### Default (no explicit path)
```
~/.local/share/ov/encrypted/
  ov-<image>-<name>/
    cipher/    # Encrypted data (always on disk)
    plain/     # Decrypted mount point (mounted on demand)
```

Override base path: `ov settings set encrypted_storage_path /path/to/storage` or `OV_ENCRYPTED_STORAGE_PATH=/path`.

## Commands

### Setup (ov config)

```bash
ov config my-app --encrypt secrets              # Set volume as encrypted + init + mount
ov config my-app --password auto                # Auto-generate password
ov config my-app --password manual              # Prompt for password
```

`ov config <image>` handles both initialization (creating cipher directories) and mounting in a single step. If volumes are already initialized, it mounts them. Password is cached in kernel keyring for multi-volume images.

### Mount

```bash
ov config mount my-app                    # Mount all encrypted volumes
ov config mount my-app --volume secrets   # Mount specific volume
```

Prompts for password (or reuses from keyring). Each volume is mounted inside a transient systemd scope unit (`ov-enc-<image>-<volume>.scope`) via `systemd-run --scope --user`. The plain directory becomes available for container use. Scope units can be listed with `systemctl --user list-units 'ov-enc-*'`.

### Unmount

```bash
ov config unmount my-app                    # Unmount all
ov config unmount my-app --volume secrets   # Unmount specific
```

Unmount calls `fusermount3 -u` then stops the scope unit (`systemctl --user stop ov-enc-<image>-<volume>.scope`) to clean up the gocryptfs daemon.

### Status

```bash
ov config status my-app
# secrets: mounted
# configs: not mounted
```

### Change Password

```bash
ov config passwd my-app
```

Changes the gocryptfs password for all encrypted volumes of an image.

## Single Password

When an image has multiple encrypted volumes, `ov config`, `ov config mount`, and `ov start` all use `systemd-ask-password --id=ov-<image>` to cache the passphrase in the kernel keyring. Password is prompted once and reused for all volumes.

## Credential lookup — iteration-capable `ssClient`

**Historical problem:** the `zalando/go-keyring` library `ov` originally
depended on looks up credentials via the Secret Service `default` alias
**only** — no iteration. If that alias resolved to a broken or stub
collection (commonly seen with KeePassXC's FdoSecrets plugin when a
previously-exposed database is unloaded), every credential lookup failed,
and `ov config mount` would hang forever polling for a keyring that couldn't
serve the secret. The polling loop had no deadline, and quadlet units with
`TimeoutStartSec=0` would wedge in `activating (start-pre)` indefinitely.

**Since 2026-04,** `ov` ships its own minimal godbus-based Secret Service
client (`ov/secret_service.go`, the `ssClient` type) with a three-step
iteration in `findItemAcrossCollections`:

1. Try the collection at the `default` alias (if healthy — skipped if its
   property reads error out).
2. Try a collection matching the `keyring_collection_label` setting (if
   non-empty — see `/ov:settings`). Useful for pinning a specific
   collection by label in multi-database setups.
3. Try every other healthy collection in listing order.

Broken collections are skipped with a diagnostic line to stderr:

    ov: skipping broken Secret Service collection <path>: <error>
    ov: Secret Service default alias target <path> is unhealthy; falling back to collection iteration

The client returns `ErrSSNotFound` if no collection has the credential, and
`ErrSSAllBroken` if every candidate errored on unlock or search (distinct
from "credential simply not stored").

**Call path:**

    KeyringStore.Get
      → keyringGetViaSSClient
        → newSSClient()
        → ssClient.findItemAnyCollection(service, username, preferLabel)
          → readAlias("default") → health check → candidate
          → collections() → filter by preferLabel → candidate
          → collections() → filter healthy, dedup vs already-tried → candidates
          → for each candidate: unlock → SearchItems({service, username}) → first match wins
        → ssClient.getSecret(item) → []byte

**Source:** `ov/secret_service.go` (ssClient + findItemAcrossCollections),
`ov/credential_keyring.go:Get` (the KeyringStore.Get entry point that
delegates to ssClient). Covered by 9 unit tests in
`ov/secret_service_test.go` including default-alias-healthy,
default-alias-broken-fallback-to-iteration, preferLabel routing, all-broken
→ ErrSSAllBroken, not-found-anywhere → ErrSSNotFound, search/unlock errors,
and candidate dedup.

## Credential source semantics

`ResolveCredential` at `ov/credential_store.go:139` returns a `(value, source)`
pair where the source string is one of:

| Source | Meaning | Caller reaction |
|---|---|---|
| `env`         | Resolved from an env var override (e.g. `GOCRYPTFS_PASSWORD`) | Terminal: use the value |
| `keyring`     | Found in the system keyring via the iteration-capable read path | Terminal: use the value |
| `kdbx`        | Found in the configured KeePass file | Terminal: use the value |
| `config`      | Found in `~/.config/ov/config.yml` (fallback or explicit backend) | Terminal: use the value |
| `locked`      | Primary backend is present but locked (e.g. keyring not yet unlocked after login) | **Retry** — backend may unlock shortly |
| `unavailable` | Primary backend probe failed (e.g. ssClient saw every collection error out); fell back to `ConfigFileStore` but the credential isn't stored there either | **Retry with backoff** — may be transient at early boot |
| `default`     | Backend queried successfully and the credential is **not stored anywhere** | **Terminal** — prompt the user interactively or fail with remediation |

**The critical distinction** is between `default` and `unavailable`: both
look identical from the ConfigFileStore return value (empty string, nil
error), but they have opposite recovery semantics. The previous ov code
conflated them and polled forever on both, which is why a broken keyring
used to wedge `ov-<image>.service` under `TimeoutStartSec=0`.

**Under systemd (`INVOCATION_ID` set, typical for `ExecStartPre`):**

- `default` → **fail immediately** with an actionable error ("encryption
  passphrase not stored for ov/enc/<image>; store with `ov secrets set ov/enc
  <image>`, or switch backend with `ov config set secret_backend config|kdbx`").
- `locked` / `unavailable` → **retry** up to `encMountDeadline` (package
  variable in `ov/enc.go`, default `2 * time.Minute`, poll period
  `encMountPollPeriod = 5 * time.Second`). After the deadline elapses, fail
  with a diagnostic listing backend, source, and remediation.
- `env` / `keyring` / `kdbx` / `config` with a non-empty value → return
  immediately.

**In interactive mode** (no `INVOCATION_ID`), `resolveEncPassphraseForMount`
delegates to `resolveEncPassphrase` which prompts via the extpass script on
the controlling TTY.

Covered by table tests in `ov/enc_resolve_mount_test.go` (7 cases including
`default` fails fast, `locked` retries until deadline, `unavailable` retries
until deadline, `keyring`/`config` with values return immediately, explicit
non-keyring backend fails fast without polling, reset callback invocation).

## Troubleshooting: broken Secret Service collection

**Symptom:** `ov config mount <image>` hangs, or a service's `ExecStartPre`
phase blocks in `activating (start-pre)` state. `journalctl --user -u
ov-<image>.service` shows lines like:

    ov-<image>[...]: ov: Secret Service default alias target <path> is unhealthy; falling back to collection iteration

Or `ov doctor` reports:

    [!] Secret Service collections -- N healthy + 1 broken. Broken: /org/freedesktop/secrets/collection/<path>. Healthy: "<label>"
    [!] Keyring index consistency -- <N indexed, <M> missing: ov/enc/<image>...

**Cause:** KeePassXC's FdoSecrets plugin can advertise a **stub collection**
(commonly aliased as `default`) whose every DBus method call returns
`org.freedesktop.Secret.Error.NoSuchObject` or an `Input/output error`. The
real credentials are in a sibling collection with a different label. This
pattern has been observed when a KeePassXC database was previously exposed
via FdoSecrets but later unloaded or renamed, while the collection entry
lingers in KeePassXC's internal state.

**Fix (nothing to do on the `ov` side):** `ssClient` iterates past the broken
collection automatically and finds the credential in the healthy sibling.
The credential lookup just works. No configuration change needed.

**Optional cleanup (KeePassXC side):**
1. Open KeePassXC
2. Tools → Settings → Secret Service Integration → Exposed Databases
3. Review the list; remove stale entries
4. Restart KeePassXC
5. Re-run `ov doctor` — the "Secret Service collections" check should now
   report "N healthy collection(s)" with no broken count.

**Pinning a preferred collection:** if your setup has multiple healthy
collections and you want `ov` to prefer one by label (e.g. to avoid
iteration overhead), set:

    ov settings set keyring_collection_label "<collection-label>"

`findItemAnyCollection` will try that label-matched collection after the
default alias (if healthy) and before untargeted iteration. Environment
override: `OV_KEYRING_COLLECTION_LABEL`. See `/ov:settings` for the full
runtime-config interface.

**Diagnostic direct query (busctl):** to check a specific collection by
path without using ov:

```bash
# List collections
busctl --user call org.freedesktop.secrets /org/freedesktop/secrets \
       org.freedesktop.DBus.Properties Get ss \
       org.freedesktop.Secret.Service Collections

# Read the default alias target
busctl --user call org.freedesktop.secrets /org/freedesktop/secrets \
       org.freedesktop.Secret.Service ReadAlias s default

# Probe a collection's health (should return Label without error)
busctl --user call org.freedesktop.secrets <collection-path> \
       org.freedesktop.DBus.Properties Get ss \
       org.freedesktop.Secret.Collection Label
```

A healthy collection returns its label; a broken stub returns an I/O error.

## KeePass Integration

Use the `--kdbx` global flag to specify a KeePass database for password storage:

```bash
ov --kdbx ~/.config/ov/secrets.kdbx config my-app --encrypt secrets
```

## Scope Unit Architecture

Each encrypted volume is mounted via `systemd-run --scope --user --unit=ov-enc-<image>-<volume> -- gocryptfs -allow_other <cipherdir> <plaindir>`. This creates a transient systemd scope unit that:

- **Survives container stop/restart** — scope units are independent of the container service's cgroup, so `KillMode=mixed` on service stop does not kill gocryptfs
- **Keeps mounts browsable on host** — the host user can access `plain/` directories even when the container is stopped
- **Handles stale scopes** — if a scope persists from a crash, the next mount attempt stops the stale scope and retries
- **Cleans up on unmount** — `ov config unmount` calls `fusermount3 -u` then `systemctl --user stop ov-enc-<image>-<volume>.scope`

Scope unit naming: `ov-enc-<image>-<volume>.scope` (e.g., `ov-enc-immich-ml-library.scope`).

List active scopes: `systemctl --user list-units 'ov-enc-*'`

### Why `-allow_other` is Required

Rootless podman with `--userns=keep-id` creates a two-level user namespace. During container mount setup, crun runs as inner uid 0, which maps through the namespace chain to **host uid 524288** (a subordinate uid), not the FUSE mount owner (uid 1000). FUSE's kernel check rejects access. The `-allow_other` flag bypasses this check, allowing crun to bind-mount from the FUSE filesystem. gocryptfs auto-enables `default_permissions` with `-allow_other`, so kernel UNIX permission checks still apply (0700 dirs restrict access to the mount owner). Confirmed by podman issues #14488, #15314, #16350, #25894.

## Integration with Runtime

- **`ov shell`/`ov start` (direct mode)**: resolves volume backing from deploy.yml, verifies encrypted volumes are mounted, appends `-v <plain>:<container-path>` flags. `ov start` mounts encrypted volumes inline via systemd-run scopes before starting the container
- **`ov config` (quadlet mode)**: generates quadlet file with `ExecStartPre=ov config mount <image>` for encrypted services. ExecStartPre creates scope units internally — these are independent of the container service. Boot behavior is backend-gated (see below)
- **`ov remove --purge`**: removes named volumes
- **Data provisioning**: `ov config --seed` (default) provisions data from data layers into bind-backed directories after mounting encrypted volumes. Works for both bind and encrypted volume types

### Boot Behavior: Backend-Gated

| Credential Backend | Quadlet Behavior | User Action on Reboot |
|---|---|---|
| **Secret Service (keyring)** | `WantedBy=default.target` + `ExecStartPre` (waits for keyring) + `TimeoutStartSec=0` | None — auto-starts after login |
| **KeePass (.kdbx)** | `ExecStartPre` (guard) + NO `WantedBy` | `ov start <image>` (prompts for kdbx master) |
| **Config file / none** | `ExecStartPre` (guard) + NO `WantedBy` | `ov start <image>` (prompts interactively) |

**Secret Service flow on reboot:**
1. Boot → systemd starts user instance (linger) → quadlet service starts
2. ExecStartPre → `ov config mount` → keyring locked → polls every `encMountPollPeriod` (5s)
3. User logs in → PAM unlocks GNOME Keyring / KeePassXC unlocks database
4. Next poll → passphrase found → volumes mount → container starts
5. If no progress after `encMountDeadline` (2 min default) → ExecStartPre fails with a clear diagnostic listing backend, source, and remediation. The service goes to `failed` state instead of wedging forever.

**Bounded retry** (since 2026-04): the poll loop in
`resolveEncPassphraseForMount` is bounded by two package-level variables
in `ov/enc.go`:

- `encMountDeadline` — total wall-clock cap (default `2 * time.Minute`)
- `encMountPollPeriod` — interval between probes (default `5 * time.Second`)

Only `source=locked` and `source=unavailable` trigger retries.
`source=default` (credential not stored anywhere) is terminal and fails
immediately with an actionable error — no amount of retrying will conjure
a credential that was never stored. This is what fixed the
hang-forever-at-ExecStartPre regression that used to wedge
`ov-<image>.service` with `TimeoutStartSec=0`.

**Crash recovery:** FUSE mounts survive container restarts because scope
units are independent of the container service cgroup. On restart, the
`ExecStartPre=ov config mount` step hits the **fast-path short-circuit**
added in 2026-04:

    All encrypted volumes for <image> already mounted (N/N)

When every requested volume is already mounted, `encMount` at
`ov/enc.go:232-291` iterates the mount list once, finds every target is
live, and returns `nil` **without calling `resolveEncPassphraseForMount`
or touching the credential store at all**. This means a broken keyring
backend does NOT block restarts of running services — only fresh mounts
(e.g., after reboot) need the keyring. If gocryptfs crashes (scope dies),
the next `ov config mount` or `ov start` detects the stale scope, stops
it, and remounts fresh — at which point the iteration-capable ssClient
kicks in.

## Volume Backing Override

When a volume is configured as `type: encrypted` in deploy.yml, it overrides the default named volume. The Docker/Podman named volume is not created -- the gocryptfs mount is used instead.

```yaml
# layer.yml declares a volume:
volumes:
  - name: data
    path: "~/.myapp"

# deploy.yml configures it as encrypted:
volumes:
  - name: data
    type: encrypted
```

## Plain (Non-Encrypted) Bind Mounts

For comparison, plain bind mounts use `type: bind`:

```bash
ov config my-app --bind data=/mnt/nas/data    # Explicit host path
ov config my-app --bind data                   # Auto path: ~/.local/share/ov/volumes/my-app/data
```

Plain bind mounts do not use encrypted storage commands. They are direct host directory mounts.

**Source files (as of 2026-04):**

- `ov/enc.go` — `encMount` (with all-mounted short-circuit, `ov/enc.go:232-291`), `ensureEncryptedMounts`, `encUnmount`, scope unit lifecycle, `resolveEncPassphraseForMount` (bounded retry with `encMountDeadline`)
- `ov/secret_service.go` — godbus-based ssClient, `findItemAcrossCollections`, `ssOps` interface for test injection, `ErrSSNotFound` / `ErrSSAllBroken` sentinel errors
- `ov/credential_keyring.go` — `KeyringStore.Probe` (iterates collections, accepts if ≥1 healthy), `KeyringStore.Get` (delegates to `keyringGetViaSSClient`), index-divergence warning
- `ov/credential_store.go` — `DefaultCredentialStore` (tracks `defaultStoreProbeErr`), `ResolveCredential` (returns the new `"unavailable"` source distinctly from `"default"`)
- `ov/deploy.go` — `DeployVolumeConfig`, `ResolveVolumeBacking`
- `ov/runtime_config.go` — `KeyringCollectionLabel` field (the `keyring_collection_label` setting)

## Cross-References

- `/ov:deploy` -- Quadlet integration, volume backing configuration, deploy.yml
- `/ov:config` -- `encrypted_storage_path` and `volumes_path` settings, `ov config mount` short-circuit fast-path documented there too
- `/ov:service` -- Container lifecycle, `ov start` inline mount
- `/ov:secrets` -- Credential store hierarchy (env → keyring → kdbx → config), `ov secrets set ov/enc <image>` to store a gocryptfs passphrase explicitly, `ov secrets list` to inspect indexed keys
- `/ov:settings` -- `secret_backend`, `keyring_collection_label`, `encrypted_storage_path`, and other runtime config keys that control credential + volume resolution
- `/ov:doctor` -- "Secret Service collections" health check, "Keyring index consistency" cross-check; invoke `ov doctor` when diagnosing broken-collection symptoms

## When to Use This Skill

**MUST be invoked** when the task involves encrypted storage, gocryptfs, or encrypted volume backing. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Pre-deployment. Configure encrypted volumes during `ov config`. See also `/ov:deploy` (volume backing).
