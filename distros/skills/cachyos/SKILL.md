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
`docker.io/cachyos/cachyos-v3` (optimized for modern `x86_64_v3` CPUs), pinned
by digest in `image/cachyos/box.yml` — Docker Hub publishes only a `:latest`
tag for `cachyos-v3`, so a digest is the most precise pin available.
CachyOS is an Arch derivative, so it shares the Arch toolchain, `pacman`, and the
`arch-builder` multi-stage builder.

The CachyOS family lives in the **`overthinkos/cachyos`** repo (git submodule at
**`image/cachyos`**), whose config is `charly.yml` plus its per-kind sibling
files (`box.yml`/`pod.yml`/`k8s.yml`/`vm.yml`), flat-imported via `import:`. The
`cachyos` base image is **owned there**. It composes the main repo's layers by git reference, flat-imports
the shared `build.yml`, and imports the main repo under the `ov` namespace
(`import: [{ov: ../..}]`) so it reaches `ov.arch` / `ov.arch-builder`. Build it
from the submodule: `charly -C image/cachyos image build cachyos` (or
`charly --repo overthinkos/cachyos image build cachyos`).

## Image Properties

| Property | Value |
|----------|-------|
| Base | docker.io/cachyos/cachyos-v3 (pinned by digest in box.yml) |
| Layers | (none) |
| Platforms | linux/amd64 |
| Distro | cachyos, arch |
| Build | pac |
| Builders | pixi, npm, cargo, aur → arch-builder |
| Registry | ghcr.io/overthinkos |
| Home repo | overthinkos/cachyos (`image/cachyos`) |

## main ↔ cachyos coupling

The main repo's `versa` image is `base: cachyos.cachyos`. Because the cachyos
base now lives in the submodule, main's `charly.yml` mounts it under the
`cachyos` import namespace:

```yaml
# main charly.yml
import:
  - cachyos: image/cachyos        # namespaced child import → cachyos.<entry>

box:
  versa:
    base: cachyos.cachyos         # the `cachyos` image inside the `cachyos` namespace
```

This is a deliberate **main → cachyos** dependency — building `versa` on main
needs the cachyos repo reachable. The submodule, in turn, imports the main repo
under the `ov` namespace (for `ov.arch-builder`), so the two repos import each
other. That mutual import is NOT a deadlock: the loader breaks the cycle **by
repo identity, not pinned version** — cachyos's `ov:` back-reference to main
resolves to the LOCAL main working tree (registered under its `repo:` identity),
even when cachyos's published release pins an older main. So main's namespace
pins win, and a stale transitive pin inside a cachyos release never drags a
divergent main snapshot into the load (see `/charly-internals:go`
"import-namespace loader"). The image DAG
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
  `/charly-image:layer` "AUR"); the `arch` distro tag is what cachyos images match
  (their `distro:` is `[cachyos, arch]`), so the same `distro.arch` sections used
  by every Arch image apply unchanged.
- The `cachyos` base declares `produce: [pixi, npm, cargo, aur]` (identical to
  `arch`), advertising the same builder-capability profile as every other base
  distro.

There is **no cachyos-specific AUR path** and no cachyos-only builder — AUR on
cachyos and AUR on arch are the same code path through `arch-builder`.

## Quick Start

```bash
charly -C image/cachyos image build cachyos
charly shell cachyos -c "pacman --version"
```

## Derived / sibling entries (all in overthinkos/cachyos)

- `/charly-distros:cachyos-pacstrap-builder` — privileged pacstrap builder (`base: ov.arch`)
- `/charly-distros:cachyos-pacstrap` — bootstrap-from-scratch rootfs (builds end-to-end)
- `/charly-vm:cachyos` — bootstrap VM (`cachyos-vm`) + `eval-cachyos-vm` eval bed
- `/charly-local:ov-cachyos` — the operator CachyOS workstation profile
- `/charly-versa:versa` — the main-repo consumer (`base: cachyos.cachyos`)

### CachyOS GPU image family

The submodule also carries a CachyOS GPU image family — the Arch/CachyOS siblings
of the Fedora GPU images (which stay in `image/nvidia` / main). They build on the
`cachyos.nvidia` GPU base, which is `cachyos` + `agent-forwarding` + `nvidia` +
`cuda` (the `nvidia` and `cuda` layers are multi-distro — Fedora rpm + Arch pac —
so they compose unchanged on CachyOS):

- `cachyos.nvidia` — the GPU base (cachyos + agent-forwarding + nvidia + cuda)
- `cachyos.python-ml` — ML Python environment
- `cachyos.jupyter-ml` — CUDA ML JupyterLab
- `cachyos.ollama` — Ollama LLM server
- `cachyos.comfyui` — ComfyUI image generation
- `cachyos.unsloth-studio` — Unsloth Studio fine-tuning UI
- `cachyos.immich-ml` — Immich with the CUDA ML backend
- `cachyos.selkies-labwc-nvidia` — GPU NVENC Selkies streaming desktop (labwc flavor)
- `cachyos.selkies-kde-nvidia` — GPU NVENC Selkies streaming desktop (full KDE Plasma flavor)
- `cachyos.selkies-kde` — full KDE Plasma Selkies flavor on the plain cachyos base (VAAPI on an AMD/Intel render node, software x264 otherwise; the `-nvidia` sibling adds NVENC). The labwc cpu/amd flavor (`selkies-labwc`) lives in image/selkies.

## Why Docker Hub instead of pacstrap

The canonical base pulls the upstream OCI image (the path the CachyOS project
itself recommends — see https://github.com/CachyOS/docker). It's the faster
default (no privileged pacstrap, no kernel build). The pacstrap-from-scratch
variant (`/charly-distros:cachyos-pacstrap`) is retained for offline/air-gapped
builds and also builds end-to-end (the pacstrap renderer derives
`[options] Architecture` from the cachyos-v3 microarch repos and emits per-repo
`SigLevel`).

## Verification

After `charly -C image/cachyos image build cachyos`:
- `charly box list` — image appears
- `charly shell cachyos -c "pacman --version"` — pacman available
- `charly box inspect versa --format base` (from main) → `cachyos.cachyos` (the `cachyos` import namespace resolves)
- `charly eval box cachyos` — build-scope eval: 3 probes pass (os-release `ID=cachyos`,
  `pacman --version`, `pacman-conf --repo-list` contains `cachyos-v3`). These also
  pass when `cachyos` is built from main via the `cachyos` import namespace, and are
  inherited by `versa` through the base chain.

## When to Use This Skill

**MUST be invoked** when the task involves the cachyos base image, the
overthinkos/cachyos submodule, or the main↔cachyos import-namespace coupling.
Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/charly-distros:arch` — the Arch base (`cachyos-pacstrap-builder` is `base: ov.arch`, via the `ov` import namespace)
- `/charly-image:image` — image family umbrella (composition, build/validate/inspect)
- `/charly-internals:cutover-policy` — the hard-cutover policy governing submodule splits
