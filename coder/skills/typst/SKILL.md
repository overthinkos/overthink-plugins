---
name: typst
description: |
  Typst document processor binary for typesetting and document generation.
  Use when working with Typst, document compilation, or typesetting tools.
---

# typst -- document processor

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `task:` |

## Usage

```yaml
# box.yml
my-image:
  layers:
    - typst
```

## Used In Images

- `/ov-distros:bazzite`

## Related Layers
- `/ov-tools:vscode` — editor sibling in bazzite
- `/ov-selkies:desktop-apps` — desktop tooling sibling

## Related Commands
- `/ov-core:shell` — run typst inside the container
- `/ov-build:build` — rebuild after layer changes

## When to Use This Skill

Use when the user asks about:

- Typst document processor installation
- Document typesetting or compilation
- The typst binary in containers

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
