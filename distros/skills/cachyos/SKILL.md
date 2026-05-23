---
name: cachyos
description: |
  CachyOS base image (docker.io/cachyos/cachyos-v3) — x86_64_v3-optimized Arch
  derivative. Owned by the overthinkos/cachyos submodule (image/cachyos);
  consumed by main's versa image via the `cachyos` import namespace.
  MUST be invoked before building, deploying, or troubleshooting cachyos images.
---

# cachyos

CachyOS base image, pulled from the upstream-published OCI image
`docker.io/cachyos/cachyos-v3:latest` (optimized for modern `x86_64_v3` CPUs).
CachyOS is an Arch derivative, so it shares the Arch toolchain, `pacman`, and the
`arch-builder` multi-stage builder.

The CachyOS family lives in the **`overthinkos/cachyos`** repo (git submodule at
**`image/cachyos`**), which is a single `overthink.yml` (all per-kind entries
inlined). The `cachyos` base image is **owned there**, inline in that
`overthink.yml`. It composes the main repo's layers by git reference, flat-imports
the shared `build.yml`, and imports the main repo under the `ov` namespace
(`import: [{ov: ../..}]`) so it reaches `ov.arch` / `ov.arch-builder`. Build it
from the submodule: `ov -C image/cachyos image build cachyos` (or
`ov --repo overthinkos/cachyos image build cachyos`).

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

The main repo's `versa` image is `base: cachyos.cachyos`. Because the cachyos
base now lives in the submodule, main's `overthink.yml` mounts it under the
`cachyos` import namespace:

```yaml
# main overthink.yml
import:
  - cachyos: image/cachyos        # namespaced child import → cachyos.<entry>

image:
  versa:
    base: cachyos.cachyos         # the `cachyos` image inside the `cachyos` namespace
```

This is a deliberate **main → cachyos** dependency — building `versa` on main
needs the cachyos repo reachable. The submodule, in turn, imports the main repo
under the `ov` namespace (for `ov.arch-builder`), so the two repos import each
other. That mutual import is NOT a deadlock: the loader records an in-progress
namespace node in its shared `nsCache` BEFORE recursing, breaking the cycle (see
`/ov-internals:go` "import-namespace loader"). The image DAG
`versa → cachyos → docker.io/cachyos-v3` is itself acyclic. `versa` inherits its
`distro:`/`build:` (values) from this base across the namespace boundary; its
`builder:` map (pixi/npm/cargo/aur → `arch-builder`) is namespace-relative, so
`versa` declares its own qualified builder map rather than inheriting one across
the boundary.

## AUR support — full parity with `arch`

CachyOS has the **same AUR capability as the `arch` base**, because it is
Arch-derived and its `builder.aur` points at the shared `arch-builder` (which
ships `yay`). Anything that builds on arch builds on cachyos:

- An image based on `cachyos` that needs AUR packages declares
  `build: [pac, aur]` (exactly as an `arch`-based image would — the base itself
  declares only `build: [pac]`, so the consumer opts in). The AUR builder stage
  (`<layer>-aur-build` via `arch-builder`) then compiles the packages and
  `pacman -U`-installs the `.pkg.tar.zst` artifacts. Worked example: the
  `selkies-desktop` image (`base: cachyos.cachyos`, `build: [pac, aur]`) builds
  `google-chrome` (chrome layer) + `wlrctl` (wl-tools layer) from the AUR.
- Layers author AUR packages under `distro.arch.aur.package` (see
  `/ov-image:layer` "AUR"); the `arch` distro tag is what cachyos images match
  (their `distro:` is `[cachyos, arch]`), so the same `distro.arch` sections used
  by every Arch image apply unchanged.
- The `cachyos` base declares `produce: [pixi, npm, cargo, aur]` (identical to
  `arch`), advertising the same builder-capability profile as every other base
  distro.

There is **no cachyos-specific AUR path** and no cachyos-only builder — AUR on
cachyos and AUR on arch are the same code path through `arch-builder`.

## Quick Start

```bash
ov -C image/cachyos image build cachyos
ov shell cachyos -c "pacman --version"
```

## Derived / sibling entries (all in overthinkos/cachyos)

- `/ov-distros:cachyos-pacstrap-builder` — privileged pacstrap builder (`base: ov.arch`)
- `/ov-distros:cachyos-pacstrap` — bootstrap-from-scratch rootfs (builds end-to-end)
- `/ov-vm:cachyos` — bootstrap VM (`cachyos-vm`) + `cachyos-vm-deploy` eval bed
- `/ov-local:ov-cachyos` — the operator CachyOS workstation profile
- `/ov-versa:versa` — the main-repo consumer (`base: cachyos.cachyos`)

## Why Docker Hub instead of pacstrap

The canonical base pulls the upstream OCI image (the path the CachyOS project
itself recommends — see https://github.com/CachyOS/docker). It's the faster
default (no privileged pacstrap, no kernel build). The pacstrap-from-scratch
variant (`/ov-distros:cachyos-pacstrap`) is retained for offline/air-gapped
builds and also builds end-to-end (the pacstrap renderer derives
`[options] Architecture` from the cachyos-v3 microarch repos and emits per-repo
`SigLevel`).

## Verification

After `ov -C image/cachyos image build cachyos`:
- `ov image list` — image appears
- `ov shell cachyos -c "pacman --version"` — pacman available
- `ov image inspect versa --format base` (from main) → `cachyos.cachyos` (the `cachyos` import namespace resolves)
- `ov eval image cachyos` — build-scope eval: 3 probes pass (os-release `ID=cachyos`,
  `pacman --version`, `pacman-conf --repo-list` contains `cachyos-v3`). These also
  pass when `cachyos` is built from main via the `cachyos` import namespace, and are
  inherited by `versa` through the base chain.

## When to Use This Skill

**MUST be invoked** when the task involves the cachyos base image, the
overthinkos/cachyos submodule, or the main↔cachyos import-namespace coupling.
Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-distros:arch` — the Arch base (`cachyos-pacstrap-builder` is `base: ov.arch`, via the `ov` import namespace)
- `/ov-image:image` — image family umbrella (composition, build/validate/inspect)
- `/ov-internals:cutover-policy` — the hard-cutover policy governing submodule splits
