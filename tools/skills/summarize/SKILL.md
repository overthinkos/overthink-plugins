---
name: summarize
description: |
  Summarize CLI for extracting text/transcripts from URLs and files.
  Use when working with the summarize layer.
---

# summarize -- URL/transcript extractor CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# image.yml or layer.yml
layers:
  - summarize
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:nodejs` — runtime dependency
- `/ov-openclaw:openclaw-full` — metalayer that bundles summarize
- `/ov-tools:whisper` — speech-to-text companion in the `openclaw-full-ml` layer

## Related Commands
- `/ov-core:shell` — run summarize CLI inside the container
- `/ov-build:build` — rebuild after layer changes

## When to Use This Skill

Use when the user asks about:
- Extracting text or transcripts from URLs
- Content summarization tools
- The `summarize` layer

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
