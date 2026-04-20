---
name: language-runtimes
description: |
  Multi-language runtime meta-layer — Go, PHP, .NET 9 SDK, nodejs-devel,
  python3-devel, ramalama. System Python via RPM (not pixi-python). Uses
  nodejs and rust layers as explicit deps.
  Use when working with polyglot development or composing multiple language
  runtimes into a single image.
---

# language-runtimes -- Go, PHP, .NET, nodejs-devel, python3-devel

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs`, `rust` |
| Install files | `layer.yml` (packages only) |

## Packages (RPM)

- `dotnet-sdk-9.0` — Microsoft .NET 9 SDK (~600 MB, dominant size)
- `golang-bin` — Go compiler + standard library
- `golang-bazil-fuse-devel` — Go FUSE bindings (for layers compiling FUSE Go code)
- `libicu` — ICU i18n library (required by .NET)
- `php` — PHP CLI + core modules
- `python3-devel` — system Python 3 + dev headers
- `python3-ramalama` — RamaLama tool (Python)
- `nodejs-devel` — Node.js headers (some native-module compiles need this alongside the `nodejs` layer's runtime)

## The vestigial `depends: python` removed in 2026-04

The layer used to declare `depends: python`, pulling in the
`python` ov-layer → `pixi` ov-layer → a conda-forge Python env
(~500 MB). But this layer installs `python3-devel` + `python3-ramalama`
via RPM — **system Python**. The pixi-python env was never referenced
by anything this layer installs. The dep was dropped in 2026-04;
consumers of `language-runtimes` now get only the RPM Python stack.

Consequence for `/ov-images:fedora-coder` (the biggest consumer): the
whole `python` / `pixi` ov-layer chain drops out of the resolved layer
set (since `/ov-layers:uv` also dropped its vestigial python dep and
`/ov-layers:supervisord` also removed its). Image size dropped by
several hundred MB. See CLAUDE.md "Key Rules" → *"Don't declare
defensive deps"* for the general rule.

If you genuinely need the pixi-python env (e.g. a layer that
installs a Python package from conda-forge via pixi), declare
`depends: python` on THAT layer directly — don't rely on transitive
pulls.

## Tests

Six build-scope tests ship with the layer:

| Test | Purpose |
|---|---|
| `dotnet-binary` + `dotnet-version` | .NET SDK installed and responsive |
| `php-binary` + `php-version` | PHP CLI reachable |
| `system-python3` + `system-python3-version` | `/usr/bin/python3` is the system interpreter (the RPM-installed one; explicitly NOT a pixi-env path) |

Go and Node.js testing is delegated to the `/ov-layers:golang` and
`/ov-layers:nodejs` / `/ov-layers:nodejs24` layer skills — those are
the single sources of truth for their respective binaries.

## Usage

```yaml
# image.yml
my-polyglot:
  layers:
    - language-runtimes
```

## Used In Images

- `/ov-images:fedora-coder` — kitchen-sink dev image, canonical consumer
- `/ov-images:bazzite-ai` (disabled)

## Related Layers

- `/ov-layers:nodejs` — Node.js runtime (direct dependency)
- `/ov-layers:rust` — Rust toolchain (direct dependency)
- `/ov-layers:python` — Pixi-python env. **Not a dep of this layer** as of 2026-04.
- `/ov-layers:golang` — Go toolchain (may be added separately for clarity even though `golang-bin` is already in this layer's RPM list)

## Related Commands

- `/ov:layer` — authoring reference
- `/ov:shell` — verify runtimes inside a container

## When to Use This Skill

**MUST be invoked** when:

- Composing `language-runtimes` into an image.
- Debating pixi-python vs. system python3 — this layer is the canonical
  "system python3 only" example.
- Understanding why the `python` ov-layer is missing from an image that
  uses language-runtimes (that's by design since 2026-04).
