---
name: himalaya
description: |
  Himalaya email CLI (IMAP/SMTP).
  Use when working with the himalaya layer.
---

# himalaya -- Email CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `task:` |
| Depends | `rust` |

## Usage

```yaml
# box or candy charly.yml
layers:
  - himalaya
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/charly-coder:rust` — required Rust toolchain dependency
- `/charly-openclaw:openclaw-full` — metalayer that includes himalaya
- `/charly-infrastructure:gnupg` — pairs with himalaya for PGP-encrypted email

## Related Commands
- `/charly-build:secrets` — provision IMAP/SMTP credentials for himalaya
- `/charly-core:shell` — run himalaya interactively inside a container
- `/charly-build:build` — compiles himalaya via the Cargo builder during image build

## When to Use This Skill

Use when the user asks about:
- Email CLI access (IMAP/SMTP)
- Himalaya email client in containers
- The `himalaya` layer

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
