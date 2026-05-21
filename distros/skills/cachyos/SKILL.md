---
name: cachyos
description: |
  CachyOS base image (docker.io/cachyos/cachyos-v3) — x86_64_v3-optimized Arch
  derivative. Owned by the overthinkos/cachyos submodule (image/cachyos) since
  the 2026-05 split; consumed by main's versa image via remote include.
  MUST be invoked before building, deploying, or troubleshooting cachyos images.
---

# cachyos

CachyOS base image, pulled from the upstream-published OCI image
`docker.io/cachyos/cachyos-v3:latest` (optimized for modern `x86_64_v3` CPUs).
CachyOS is an Arch derivative, so it shares the Arch toolchain, `pacman`, and the
`arch-builder` multi-stage builder.

> **Relocated (2026-05):** the CachyOS family was split out of the main repo into
> the **`overthinkos/cachyos`** repo (git submodule at **`image/cachyos`**). The
> `cachyos` base image is **owned there** (in that repo's `cachyos-base.yml`) and
> composes the main repo's layers + shared `build.yml` + `arch-base.yml` by git
> reference. Build it from the submodule:
> `ov -C image/cachyos image build cachyos` (or `ov --repo overthinkos/cachyos image build cachyos`).

## Image Properties

| Property | Value |
|----------|-------|
| Base | docker.io/cachyos/cachyos-v3:latest |
| Layers | (none) |
| Platforms | linux/amd64 |
| Distro | cachyos, arch |
| Build | pac |
| Builders | pixi, npm, cargo, aur → arch-builder |
| Registry | ghcr.io/overthinkos |
| Home repo | overthinkos/cachyos (`image/cachyos`) |

## main ↔ cachyos coupling

The main repo's `versa` image is `base: cachyos`. Because the cachyos base now
lives in the submodule, main's `overthink.yml` pulls it back by remote include:

```yaml
# main overthink.yml
include:
  - '@github.com/overthinkos/cachyos/cachyos-base.yml:<tag>'
```

This is a deliberate **main → cachyos** dependency — building `versa` on main
needs the cachyos repo reachable. It is NOT a resolution cycle (single-file
includes; the image DAG `versa → cachyos → docker.io/cachyos-v3` is acyclic).
`versa` inherits its `builder:` map (pixi/npm/cargo/aur → `arch-builder`) from
this base — it carries no per-image builder override.

## Quick Start

```bash
ov -C image/cachyos image build cachyos
ov shell cachyos -c "pacman --version"
```

## Derived / sibling entries (all in overthinkos/cachyos)

- `/ov-distros:cachyos-pacstrap-builder` — privileged pacstrap builder (`base: arch`)
- `/ov-distros:cachyos-pacstrap` — bootstrap-from-scratch rootfs (GPGME caveat)
- `/ov-vm:cachyos` — bootstrap VM (`cachyos-vm`) + `cachyos-vm-deploy` bed
- `/ov-local:ov-cachyos` — the operator CachyOS workstation profile
- `/ov-versa:versa` — the main-repo consumer (`base: cachyos`)

## Why Docker Hub instead of pacstrap

The canonical base pulls the upstream OCI image (the path the CachyOS project
itself recommends — see https://github.com/CachyOS/docker). The
pacstrap-from-scratch variant (`/ov-distros:cachyos-pacstrap`) is retained for
offline/air-gapped builds but is known to fail under sudo podman + pacman 7.x
(GPGME context init). Prefer this base.

## Verification

After `ov -C image/cachyos image build cachyos`:
- `ov image list` — image appears
- `ov shell cachyos -c "pacman --version"` — pacman available
- `ov image inspect versa --format base` (from main) → `cachyos` (remote include resolves)
- `ov eval image cachyos` — build-scope eval: 3 probes pass (os-release `ID=cachyos`,
  `pacman --version`, `pacman-conf --repo-list` contains `cachyos-v3`). These also
  pass when `cachyos` is built from main via the remote include, and are inherited
  by `versa` through the base chain.

## When to Use This Skill

**MUST be invoked** when the task involves the cachyos base image, the
overthinkos/cachyos submodule, or the main↔cachyos remote-include coupling.
Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-distros:arch` — the Arch base (`cachyos-pacstrap-builder` is `base: arch`)
- `/ov-image:image` — image family umbrella (composition, build/validate/inspect)
- `/ov-internals:cutover-policy` — the hard-cutover that split the family out
