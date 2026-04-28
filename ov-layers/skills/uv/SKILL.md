---
name: uv
description: |
  uv + uvx — Astral's fast Python package/project manager. Rewritten 2026-04
  to install as a direct-download binary (no pixi env, no Python dep). Pulled
  via the `download:` verb with `strip_components: 1` to handle the upstream
  tarball's arch-prefixed top-level directory.
  Use when working with the uv layer or when deciding whether to install a
  CLI tool via pixi vs. direct binary download.
---

# uv -- Python package / project manager (astral-sh/uv)

## Layer Properties

| Property | Value |
|----------|-------|
| Kind | Direct-download binary layer |
| Install files | `layer.yml` (no pixi.toml, no dependencies) |
| Depends | **(none)** — uv is a self-contained Rust binary |
| Binaries | `/usr/local/bin/uv`, `/usr/local/bin/uvx` |

## Why direct download (not pixi)

Pre-2026-04 the layer shipped a `pixi.toml` with `uv = "*"` and
`depends: python`. That installed uv into `$HOME/.pixi/envs/default/bin/`
as part of a conda-forge Python environment — despite uv being a
completely self-contained Rust binary needing no Python runtime. The
side effects:

- Image bloat — the pixi env drags in ~500 MB of Python stdlib + numpy
  transitively, nothing of which uv actually uses.
- HOME-dependency — the binary path varied per-image's `$HOME`, and
  HOME-relative PATH entries in child images got subtly wrong when the
  `ov` auto-intermediate machinery baked uid=1000 paths into images
  deployed at uid=0 (fixed in `ov/intermediates.go` — see
  `/ov-dev:generate` "UID-keyed sibling grouping").
- Confusion — "uv is installed but `which uv` says not found" for
  anyone who wasn't in a pixi shell.

The rewrite installs uv the way `/ov-layers:typst` and
`/ov-layers:pixi` itself already did — fetch the upstream binary,
unpack to `/usr/local/bin`, done. System-wide reachable, always on
PATH, no HOME gymnastics.

## Install task

```yaml
tasks:
  - download: "https://github.com/astral-sh/uv/releases/latest/download/uv-${BUILD_ARCH}-unknown-linux-gnu.tar.gz"
    extract: tar.gz
    strip_components: 1
    to: /usr/local/bin
    user: root
```

`${BUILD_ARCH}` expands to `x86_64` / `aarch64` at build time. Upstream
ships per-arch tarballs under stable latest-download URLs, nested one
level deep (`uv-x86_64-unknown-linux-gnu/uv`, `…/uvx`). The
`strip_components: 1` modifier (new in 2026-04, see `/ov:layer`
"Download verb") collapses that wrapper so both binaries land
directly at `/usr/local/bin/uv` and `/usr/local/bin/uvx`.

## Tests

Three build-scope tests ship with the layer:

| Test | Purpose |
|---|---|
| `uv-binary` | `/usr/local/bin/uv` exists |
| `uv-version` | `uv --version` exits 0 |
| `uvx-binary` | `/usr/local/bin/uvx` exists (the per-run tool runner) |

## Used In Images

- `/ov-images:fedora-coder` — the canonical kitchen-sink dev image.
- Any image composing `hermes-full` or directly including `uv`.

## Related Layers

- `/ov-layers:pixi` — direct-download pattern this layer now mirrors. Pixi remains a legitimate multi-binary Python env manager for images that genuinely need one (jupyter, whisper, openwebui); uv is too small to justify the overhead.
- `/ov-layers:typst` — sibling direct-download binary pattern.
- `/ov-layers:python` — the pixi-python meta-layer this layer **no longer** depends on.
- `/ov-layers:rust` — system Rust toolchain, if you want to build uv from source instead (`cargo install uv`). Usually not worth it — the Astral binary is pre-optimized.

## Related Commands

- `/ov:layer` — authoring reference (covers `download:` verb + `strip_components:` modifier)
- `/ov:shell` — run `uv` inside a container

## When to Use This Skill

**MUST be invoked** when:

- Adding `uv` to an image's layer list.
- Authoring a sibling layer for a small Rust/Go CLI tool — this is the
  canonical pattern (direct download > pixi > anything else).
- Debugging why `uv --version` fails: it's a `$PATH` or binary-landing-
  at-wrong-path issue, not a Python env problem.
- Understanding why "uv has no dependencies" is correct even though it
  manages Python packages.

## Related

- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
