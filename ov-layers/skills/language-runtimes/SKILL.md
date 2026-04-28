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

## Packages (pac) — Arch Linux

- `dotnet-sdk` — in `[extra]` (no third-party repo needed)
- `go`, `icu`, `php`, `python` — all `[core]` / `[extra]`. Arch ships headers with the main package (no `-devel` split), so `nodejs-devel` / `python3-devel` have no separate `pac:` entries.

Drops on Arch: `python3-ramalama` (not packaged — install via `uv tool install ramalama`), `golang-bazil-fuse-devel` (Go library fetched via `go get`).

## Packages (deb) — Debian + Ubuntu

- `golang-go` — Go compiler + stdlib
- `libicu-dev` — ICU with dev headers
- `php-cli` — PHP CLI binary (Debian splits `php` into `php-cli` / `php-fpm` / etc.)
- `python3-dev` — system Python 3 + dev headers

`dotnet-sdk-9.0` is **not** in the `deb:` package list. It's installed by a cross-distro `cmd:` task via Microsoft's official `dotnet-install.sh` — see the next subsection. `python3-ramalama`, `golang-bazil-fuse-devel`, `nodejs-devel` are dropped (not packaged on Debian/Ubuntu).

### dotnet-sdk-9.0 via Microsoft's `dotnet-install.sh` (2026-04)

Cross-distro parity for .NET 9 requires juggling three asymmetric availability windows:

| Distro | Where dotnet-sdk-9.0 lives |
|---|---|
| Fedora 43 | Distro repo (`rpm:` pulls `dotnet-sdk-9.0`) |
| Arch | `[extra]` (`pac:` pulls `dotnet-sdk`) |
| Debian 13 trixie | Microsoft's trixie apt repo has it; Debian main does not |
| Ubuntu 24.04 noble | **Neither** Canonical noble (ships 8.0 + 10.0) **nor** Microsoft's noble apt repo (ships only 10.0) has 9.0 |

Rather than carrying that asymmetry in layer code, the layer uses Microsoft's official cross-distro installer script, channel-pinned to `9.0`. It installs to `/usr/share/dotnet` and symlinks `/usr/bin/dotnet`:

```yaml
tasks:
  - cmd: |
      if command -v dotnet >/dev/null 2>&1; then
        exit 0                        # already installed by distro rpm/pac
      fi
      if ! command -v apt-get >/dev/null 2>&1; then
        exit 0                        # non-Debian-family + no dotnet — intentional drop
      fi
      install -d /usr/share/dotnet
      curl -fsSL https://builds.dotnet.microsoft.com/dotnet/scripts/v1/dotnet-install.sh -o /tmp/dotnet-install.sh
      chmod +x /tmp/dotnet-install.sh
      /tmp/dotnet-install.sh --channel 9.0 --install-dir /usr/share/dotnet
      ln -sf /usr/share/dotnet/dotnet /usr/bin/dotnet
      rm -f /tmp/dotnet-install.sh
    user: root
```

Verified 2026-04-20 on `ghcr.io/overthinkos/ubuntu-coder:latest`:

```
$ /usr/bin/dotnet --version
9.0.313
```

Idempotent: the outer `command -v dotnet` guard makes this task a no-op on Fedora/Arch (where the distro package already installed dotnet) and on rebuilds (where the previous run already placed the symlink). Runtime dependency: `libicu-dev` (already installed by the layer's `deb:` section).

See `/ov:layer` for general cross-distro task-authoring patterns.

## Tag-section overrides

- `debian:13:` — just packages (same as generic `deb:`).
- `ubuntu:24.04:` — just packages (same as generic `deb:`).

Both exist so future Microsoft-apt-repo-based installs can be slotted into tag sections via `repos:` + `packages:` without disturbing the generic `deb:` fallback.

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

- `/ov-images:fedora-coder` — kitchen-sink dev image, canonical RPM consumer.
- `/ov-images:arch-coder` — pacman-based sibling.
- `/ov-images:debian-coder`, `/ov-images:ubuntu-coder` — deb-based siblings, consumers of the dotnet-install.sh task.
- `/ov-images:bazzite-ai` (disabled).

## Related Layers

- `/ov-layers:nodejs` — Node.js runtime (direct dependency)
- `/ov-layers:rust` — Rust toolchain (direct dependency)
- `/ov-layers:python` — Pixi-python env. **Not a dep of this layer** as of 2026-04.
- `/ov-layers:golang` — Go toolchain (may be added separately for clarity even though `golang-bin` is already in this layer's RPM list)
- `/ov-layers:uv` — direct-download Rust binary (also dropped its vestigial pixi-python dep in 2026-04)

## Related Commands

- `/ov:layer` — authoring reference (tag-section cascade, `cmd:` vs declarative repos)
- `/ov:build` — `base_user:` and bootstrap packages
- `/ov:shell` — verify runtimes inside a container

## When to Use This Skill

**MUST be invoked** when:

- Composing `language-runtimes` into an image.
- Debating pixi-python vs. system python3 — this layer is the canonical
  "system python3 only" example.
- Understanding why the `python` ov-layer is missing from an image that
  uses language-runtimes (that's by design since 2026-04).

## Related

- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
