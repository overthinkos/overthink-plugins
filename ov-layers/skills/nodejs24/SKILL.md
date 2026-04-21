---
name: nodejs24
description: |
  Node.js 24 and npm via Fedora RPM packages with global npm prefix.
  Use when working with Node.js 24 or applications requiring a newer Node.js version.
---

# nodejs24 -- Node.js 24 runtime

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | none (explicit `depends: []`) |
| Install files | `layer.yml`, `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NPM_CONFIG_PREFIX` | `~/.npm-global` |
| `npm_config_cache` | `~/.cache/npm` |

PATH additions: `~/.npm-global/bin`

## Packages

RPM: `nodejs24`, `nodejs24-npm`

## Build Setup

- `tasks:` symlink verbs create (`node-24` → `node`, `npm-24` → `npm`, `npx-24` → `npx` in `/usr/local/bin`) since Fedora's parallel-installable nodejs24 package installs binaries with version suffixes
- **package.json** — Declares `pnpm@10` as a dependency. Installed via the builder system's npm pattern (runs as UID 1000, uses `/tmp/npm-cache`), NOT via `npm install -g` in tasks:. This avoids creating root-owned files in `~/.cache`

## Usage

```yaml
# image.yml
my-image:
  layers:
    - nodejs24
```

## Used In Images

- `/ov-images:immich`
- `/ov-images:immich-ml`

## Related Layers

- `/ov-layers:nodejs` -- alternative standard Node.js version

## When to Use This Skill

Use when the user asks about:

- Node.js 24 specifically
- Immich or applications needing Node.js 24
- The `nodejs24` layer

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov image test`, `ov test`)
