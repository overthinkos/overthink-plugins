---
name: cachyos-pacstrap
description: |
  Bootstrap-from-scratch CachyOS rootfs via pacstrap inside a privileged builder.
  Retained for offline/air-gapped builds; known to fail under sudo podman +
  pacman 7.x (GPGME). Lives in the overthinkos/cachyos submodule (image/cachyos).
  MUST be invoked before building or troubleshooting cachyos-pacstrap.
---

# cachyos-pacstrap

Bootstrap-from-scratch CachyOS root filesystem, built via `pacstrap` inside the
privileged `/ov-distros:cachyos-pacstrap-builder` container
(`from: builder:pacstrap`, `bootstrap_builder_image: cachyos-pacstrap-builder`).

> **Lives in `overthinkos/cachyos`** (git submodule at `image/cachyos`). Build:
> `ov -C image/cachyos image build cachyos-pacstrap`.

## When to use it (and when not)

The canonical CachyOS base (`/ov-distros:cachyos`) pulls the upstream-published
OCI image from Docker Hub — that is the recommended path and the one the CachyOS
project itself uses. This pacstrap variant exists for **offline / air-gapped**
builds and as a worked example of the `from: builder:pacstrap` +
`bootstrap_builder_image:` pattern.

## Known limitation (pacstrap not usable)

`cachyos-pacstrap` is **not currently usable**. The remote composition + builder
build succeed; the privileged pacstrap step then fails in two pre-existing ways
(both observed during R10, both config/environment-inherent):

1. Intermittent GPGME keyring/DB-sync error — `GPGME error: No data` /
   `failed to synchronize all databases (corrupted PGP signature)`.
2. When keyring init succeeds, a CachyOS `x86_64_v3` architecture rejection —
   `package architecture is not valid`. The `cachyos-v3` repos serve
   `x86_64_v3`-optimized packages (e.g. `linux-cachyos`) but the bootstrap
   pacman.conf never sets `Architecture = x86_64_v3`.

Both originate in the shared `build.yml` cachyos distro config, proven orthogonal
to the 2026-05 submodule split via an empty `git diff main` on `build.yml` + the
pacstrap runner (`ov/build.go`, `ov/privileged_runner.go`, `ov/vm_bootstrap.go`).
The fix (an `Architecture` directive in the pacstrap pacman.conf via
`ExtraPacmanConf`, plus keyring-init hardening) is a separate upstream
enhancement. Prefer `/ov-distros:cachyos`.

## Image Properties

| Property | Value |
|----------|-------|
| From | builder:pacstrap |
| bootstrap_builder_image | cachyos-pacstrap-builder |
| Distro | cachyos, arch |
| Build | pac |
| Home repo | overthinkos/cachyos (`image/cachyos`) |

The `cachyos` distro config (pacstrap base packages, CachyOS keyring
`F3B607488DB35A47`, mirrorlist, `cachyos*` repos) lives in the main repo's
`build.yml` and is remote-included by the submodule.

## Cross-References

- `/ov-distros:cachyos` — the recommended Docker-Hub base
- `/ov-distros:cachyos-pacstrap-builder` — the privileged builder it uses
- `/ov-vm:cachyos` — the VM built via the same pacstrap path (same GPGME caveat)

## When to Use This Skill

**MUST be invoked** before building or debugging the CachyOS pacstrap rootfs.
Invoke BEFORE reading source code or launching Explore agents.
