---
name: pixi
description: |
  Pixi package manager binary with environment and PATH setup.
  Use when working with pixi, conda-forge packages, or Python environment management.
---

# pixi -- Pixi package manager

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `task:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `PIXI_CACHE_DIR` | `~/.cache/pixi` |
| `RATTLER_CACHE_DIR` | `~/.cache/rattler` |

PATH additions: `~/.pixi/bin`, `~/.pixi/envs/default/bin`

## Usage

```yaml
# charly.yml
my-image:
  candy:
    - pixi
```

## Used In Images

- `/charly-distros:fedora-builder` (direct)
- `/charly-distros:arch-builder` (direct)
- Transitive dependency via `python` / `supervisord` in most service images

## Related Layers

- `/charly-languages:python` -- depends on pixi for Python 3.13 installation
- `/charly-coder:pre-commit` -- uses pixi for pre-commit installation

## Committed `pixi.lock` → `pixi install --frozen`

Every pixi layer ships a committed `pixi.lock` next to its `pixi.toml`. When a
lock is present, generation auto-flips the build stage from `pixi install` (a
full SAT solve over the conda + PyPI indexes on every cache miss) to
`pixi install --frozen` (install straight from the lock — no solve,
deterministic, better cache reuse). The flip is automatic via `HasPixiLock`
detection (`charly/layers.go`) → the `pixi.toml+lock` install command in
`build.yml`; no per-layer config.

**Regenerate the lock whenever you change `pixi.toml`** — `--frozen` fails the
build loudly if the lock is stale (no silent skew). Generate with the builder's
own pixi so the lock format matches what installs it, e.g.:

```bash
podman run --rm --userns=keep-id \
  -v "$PWD/candy/<name>:/m" -w /m \
  ghcr.io/overthinkos/fedora-builder:<calver> \
  bash -c 'grep -q system-requirements pixi.toml || printf "\n[system-requirements]\nlibc = { family = \"glibc\", version = \"2.39\" }\n" >> pixi.toml; pixi lock'
```

The `[system-requirements]` glibc fix mirrors what the build stage injects, so
the committed lock resolves the same manylinux wheels the build installs.

## When to Use This Skill

Use when the user asks about:

- Pixi package manager setup
- Conda-forge package installation
- Python environment base layer
- The `pixi` layer or `PIXI_CACHE_DIR`

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
