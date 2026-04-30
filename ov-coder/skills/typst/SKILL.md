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
| Install files | `tasks:` |

## Usage

```yaml
# image.yml
my-image:
  layers:
    - typst
```

## Used In Images

- `/ov-foundation:bazzite-ai` (disabled)

## Related Layers
- `/ov-foundation:vscode` — editor sibling in bazzite-ai
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

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
