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
| Install files | `layer.yml` (packages only) |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NPM_CONFIG_PREFIX` | `~/.npm-global` |
| `npm_config_cache` | `~/.cache/npm` |

PATH additions: `~/.npm-global/bin`

## Packages

RPM: `nodejs`, `npm`
DEB: `nodejs`, `npm`

## Usage

```yaml
# image.yml
my-image:
  layers:
    - nodejs
```

## Used In Images

- `/ov-distros:fedora-builder`
- `/ov-distros:archlinux-builder`
- `/ov-distros:fedora-remote` (via remote layer ref)
- Transitive dependency for `openclaw`, `claude-code`, `pre-commit`, and other layers

## Related Layers

- `/ov-coder:nodejs24` -- alternative Node.js 24 version
- `/ov-coder:claude-code` -- depends on nodejs
- `/ov-coder:pre-commit` -- depends on nodejs

## When to Use This Skill

Use when the user asks about:

- Node.js or npm setup in containers
- Global npm package installation
- The `nodejs` layer or npm configuration

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
