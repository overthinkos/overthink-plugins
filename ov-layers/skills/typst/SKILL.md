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
| Install files | `root.yml` |

## Usage

```yaml
# images.yml
my-image:
  layers:
    - typst
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:vscode` — editor sibling in bazzite-ai
- `/ov-layers:desktop-apps` — desktop tooling sibling

## Related Commands
- `/ov:shell` — run typst inside the container
- `/ov:build` — rebuild after layer changes

## When to Use This Skill

Use when the user asks about:

- Typst document processor installation
- Document typesetting or compilation
- The typst binary in containers
