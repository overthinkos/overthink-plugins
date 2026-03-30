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
| Install files | `layer.yml`, `root.yml` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NPM_CONFIG_PREFIX` | `~/.npm-global` |
| `npm_config_cache` | `~/.cache/npm` |

PATH additions: `~/.npm-global/bin`

## Packages

RPM: `nodejs24`, `nodejs24-npm`

## Usage

```yaml
# images.yml
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
