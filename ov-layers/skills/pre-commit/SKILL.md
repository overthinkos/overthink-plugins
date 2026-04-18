---
name: pre-commit
description: |
  Pre-commit git hooks framework via pixi, plus markdownlint-cli via npm.
  Use when working with git hooks, linting, or code quality tooling.
---

# pre-commit -- Git hooks framework

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs` |
| Install files | `layer.yml`, `pixi.toml`, `package.json`, `tasks:` |

## Packages

Pixi (conda-forge): `pre-commit`
npm: `markdownlint-cli`

## Usage

```yaml
# image.yml
my-dev:
  layers:
    - pre-commit
```

## Used In Images

- No enabled images use this layer directly (dev tool layer)

## Related Layers

- `/ov-layers:nodejs` -- required dependency (provides npm for markdownlint-cli)
- `/ov-layers:pixi` -- transitive (provides pixi for pre-commit installation)

## When to Use This Skill

Use when the user asks about:

- Pre-commit hooks in containers
- Git hook frameworks
- Markdown linting
- The `pre-commit` layer

## Author + Test References

- `/ov:layer` — layer authoring reference (tasks, vars, env_provides, tests block syntax)
- `/ov:test` — declarative testing framework for the `tests:` block
