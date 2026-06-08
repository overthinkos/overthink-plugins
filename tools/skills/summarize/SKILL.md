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
| Install files | `candy.yml`, `package.json` |
| Depends | `nodejs` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - summarize
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:nodejs` — runtime dependency
- `/charly-openclaw:openclaw-full` — metalayer that bundles summarize
- `/charly-tools:whisper` — speech-to-text companion in the `openclaw-full-ml` layer

## Related Commands
- `/charly-core:shell` — run summarize CLI inside the container
- `/charly-build:build` — rebuild after layer changes

## When to Use This Skill

Use when the user asks about:
- Extracting text or transcripts from URLs
- Content summarization tools
- The `summarize` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
