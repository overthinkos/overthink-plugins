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
| Install files | `candy.yml` (packages only) |
| Depends | none |

## Packages

RPM: `sqlite`

## Usage

```yaml
# box.yml or candy.yml
layers:
  - sqlite
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:dev-tools` -- common dev CLI bundle that pairs with sqlite
- `/charly-openclaw:openclaw-full` -- parent metalayer that bundles sqlite

## Related Commands
- `/charly-core:shell` -- run sqlite3 inside the container
- `/charly-core:cmd` -- one-shot sqlite query in a running service

## When to Use This Skill

Use when the user asks about:
- SQLite database in containers
- SQLite CLI tools
- The `sqlite` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
