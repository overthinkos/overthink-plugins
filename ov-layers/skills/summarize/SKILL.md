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
# images.yml or layer.yml
layers:
  - summarize
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- Extracting text or transcripts from URLs
- Content summarization tools
- The `summarize` layer
