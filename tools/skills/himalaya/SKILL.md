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
| Install files | `candy.yml`, `task:` |
| Depends | `rust` |

## Usage

```yaml
# box.yml or candy.yml
layers:
  - himalaya
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Layers
- `/ov-coder:rust` — required Rust toolchain dependency
- `/ov-openclaw:openclaw-full` — metalayer that includes himalaya
- `/ov-infrastructure:gnupg` — pairs with himalaya for PGP-encrypted email

## Related Commands
- `/ov-build:secrets` — provision IMAP/SMTP credentials for himalaya
- `/ov-core:shell` — run himalaya interactively inside a container
- `/ov-build:build` — compiles himalaya via the Cargo builder during image build

## When to Use This Skill

Use when the user asks about:
- Email CLI access (IMAP/SMTP)
- Himalaya email client in containers
- The `himalaya` layer

## Related

- `/ov-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval box`, `ov eval live`)
