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
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

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

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
