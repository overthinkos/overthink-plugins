---
name: sqlite
description: |
  SQLite database CLI.
  Use when working with the sqlite layer.
---

# sqlite -- SQLite database CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |
| Depends | none |

## Packages

RPM: `sqlite`

## Usage

```yaml
# image.yml or layer.yml
layers:
  - sqlite
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:dev-tools` -- common dev CLI bundle that pairs with sqlite
- `/ov-openclaw:openclaw-full` -- parent metalayer that bundles sqlite

## Related Commands
- `/ov-core:shell` -- run sqlite3 inside the container
- `/ov-core:cmd` -- one-shot sqlite query in a running service

## When to Use This Skill

Use when the user asks about:
- SQLite database in containers
- SQLite CLI tools
- The `sqlite` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
