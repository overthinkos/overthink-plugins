---
name: nodejs
description: |
  Node.js and npm via system packages (RPM/DEB) with global npm prefix.
  Use when working with Node.js, npm, or JavaScript/TypeScript tooling.
---

# nodejs -- Node.js and npm runtime

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `candy.yml`, `task:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NPM_CONFIG_PREFIX` | `~/.npm-global` |
| `npm_config_cache` | `~/.cache/npm` |

PATH additions: `~/.npm-global/bin`

## Packages

RPM: `nodejs`, `npm`
DEB: `nodejs`, `npm`
PAC: `nodejs`, `npm`

## pnpm (standalone binary)

The layer installs the **pnpm** standalone binary (self-contained — it bundles
its own node) to `/usr/local/bin/pnpm` via a `task:` download. It is deliberately
NOT a `package.json` dependency: a `package.json` would trigger the npm
multi-stage builder on this layer, and the builder images that COMPOSE nodejs
(`arch-builder` / `fedora-builder`) cannot self-provide the npm builder. The
binary on `/usr/local/bin` is on the default system PATH for every user (incl.
root — Immich runs its pnpm build as root). Immich and other pnpm consumers use
this pnpm to drive their build.

## Usage

```yaml
# box.yml
my-image:
  layers:
    - nodejs
```

## Used In Images

- `/charly-distros:fedora-builder`
- `/charly-distros:arch-builder`
- Transitive dependency for `openclaw`, `claude-code`, `pre-commit`, and other layers

## Related Layers

- `/charly-coder:claude-code` -- depends on nodejs
- `/charly-coder:pre-commit` -- depends on nodejs
- `/charly-immich:immich-layer` -- uses the layer's pnpm to build Immich

## When to Use This Skill

Use when the user asks about:

- Node.js or npm setup in containers
- Global npm package installation
- The `nodejs` layer or npm configuration

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
