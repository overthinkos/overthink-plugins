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
| Install files | `layer.yml`, `tasks:` |
| Depends | `rust` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - himalaya
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## Related Layers
- `/ov-layers:rust` — required Rust toolchain dependency
- `/ov-layers:openclaw-full` — metalayer that includes himalaya
- `/ov-layers:gnupg` — pairs with himalaya for PGP-encrypted email

## Related Commands
- `/ov:secrets` — provision IMAP/SMTP credentials for himalaya
- `/ov:shell` — run himalaya interactively inside a container
- `/ov:build` — compiles himalaya via the Cargo builder during image build

## When to Use This Skill

Use when the user asks about:
- Email CLI access (IMAP/SMTP)
- Himalaya email client in containers
- The `himalaya` layer
