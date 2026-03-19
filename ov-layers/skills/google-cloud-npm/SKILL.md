---
name: google-cloud-npm
description: |
  Google Cloud npm packages: firebase-tools and Gemini CLI.
  Use when working with Firebase, Gemini CLI, or GCP Node.js tooling.
---

# google-cloud-npm -- GCP npm packages

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `google-cloud`, `nodejs` |
| Install files | `package.json` |

## Packages

- `firebase-tools` (npm) -- Firebase CLI
- `@google/gemini-cli` (npm) -- Gemini CLI

## Usage

```yaml
# images.yml
my-image:
  layers:
    - google-cloud-npm
```

## Used In Images

Not currently used in any enabled images. Depends on `google-cloud` + `nodejs`.

## Related Layers

- `/ov-layers:google-cloud` -- Google Cloud SDK dependency
- `/ov-layers:nodejs` -- Node.js runtime dependency

## When to Use This Skill

Use when the user asks about:

- Firebase CLI tools or deployment
- Gemini CLI for Google AI
- GCP npm packages
