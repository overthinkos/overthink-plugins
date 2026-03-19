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
# images.yml or layer.yml
layers:
  - sqlite
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- SQLite database in containers
- SQLite CLI tools
- The `sqlite` layer
