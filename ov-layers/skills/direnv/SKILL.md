---
name: direnv
description: |
  direnv -- automatic environment variable loading from .envrc files.
  Use when working with direnv, .envrc, .secrets, or environment management.
---

# direnv -- Automatic environment variable loading

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `direnv`
PAC: `direnv`

## Usage

```yaml
# images.yml
my-image:
  layers:
    - direnv
```

Typically used as part of the `agent-forwarding` composition layer rather than directly. Also available in the heavyweight `dev-tools` layer (46 packages).

## Runtime Behavior

Provides the `direnv` binary for automatic environment variable loading from `.envrc` files. When entering a directory with `.envrc`, direnv evaluates it and exports environment variables.

The primary use case in Overthink is the `.secrets` workflow: `.envrc` calls `eval "$(ov secrets gpg env)"` which decrypts a GPG-encrypted `.secrets` file in memory and exports the variables — no plaintext on disk. No external `direnvrc` dependency needed.

## Used In Images

Part of `agent-forwarding` composition layer, used in 27 application images.

Also available in `dev-tools` layer (used in `bazzite-ai`, `fedora-remote`).

## Related Layers

- `/ov-layers:agent-forwarding` -- metalayer that includes gnupg + direnv + ssh-client
- `/ov-layers:gnupg` -- GPG tools (needed for `.secrets` decryption)
- `/ov-layers:dev-tools` -- heavyweight layer that also includes direnv (46 packages)

## Cross-References

- `/ov:secrets` -- `ov secrets gpg` commands for managing `.secrets` files, `ov secrets gpg setup` for GPG agent + KeePassXC configuration, `ov secrets gpg doctor` for health checks

## When to Use This Skill

Use when the user asks about:

- direnv or `.envrc` files
- The `direnv` layer
- Automatic environment variable loading
- The `.secrets` + direnv workflow
