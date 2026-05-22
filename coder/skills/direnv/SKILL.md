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

RPM: `direnv` · PAC: `direnv` · DEB: `direnv` — full cross-distro parity.

## Usage

```yaml
# image.yml
my-image:
  layers:
    - direnv
```

Typically used as part of the `agent-forwarding` composition layer rather than directly. Also available in the heavyweight `dev-tools` layer (46 packages).

## Runtime Behavior

Provides the `direnv` binary AND the per-shell hook installation (post-2026-05 cutover) for automatic environment variable loading from `.envrc` files.

**Shell hooks are now declarative (2026-05 cutover).** The layer carries a `shell:` block:

```yaml
shell:
  init: |
    eval "$(direnv hook ${SHELL_NAME})"   # bash/zsh/sh — POSIX-style
  fish:
    init: |
      direnv hook fish | source            # fish — different syntax
```

Container images get `/etc/profile.d/ov-direnv-<shell>.sh` and `/etc/fish/conf.d/ov-direnv.fish` emitted at `ov image build` time. `target: local` host deploys get a managed-block in `~/.bashrc` / `~/.zshrc` plus `~/.config/fish/conf.d/ov-direnv.fish` at `ov deploy add` time, only for shells the runtime probe finds. The pre-cutover bug (fish hook missing because `~/.config/fish/config.fish` was never edited) is structurally fixed.

The primary use case in Overthink is the `.secrets` workflow: `.envrc` calls `eval "$(ov secrets gpg env)"` which decrypts a GPG-encrypted `.secrets` file in memory and exports the variables — no plaintext on disk. No external `direnvrc` dependency needed.

## Used In Images

Part of `agent-forwarding` composition layer, used in 27 application images.

Also available in `dev-tools` layer (used in `bazzite`).

## Related Layers

- `/ov-distros:agent-forwarding` -- metalayer that includes gnupg + direnv + ssh-client
- `/ov-infrastructure:gnupg` -- GPG tools (needed for `.secrets` decryption)
- `/ov-coder:dev-tools` -- heavyweight layer that also includes direnv (46 packages)

## Cross-References

- `/ov-build:secrets` -- `ov secrets gpg` commands for managing `.secrets` files, `ov secrets gpg setup` for GPG agent + KeePassXC configuration, `ov secrets gpg doctor` for health checks

## When to Use This Skill

Use when the user asks about:

- direnv or `.envrc` files
- The `direnv` layer
- Automatic environment variable loading
- The `.secrets` + direnv workflow

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
