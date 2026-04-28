---
name: gnupg
description: |
  GnuPG encryption and signing tools for GPG agent forwarding.
  Use when working with GPG, encryption, signing, or the gnupg layer.
---

# gnupg -- GnuPG encryption and signing tools

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (packages only) |

## Packages

RPM: `gnupg2` · PAC: `gnupg` · DEB: `gnupg` — full cross-distro parity. Note: `gnupg` is also in the `debian` distro's bootstrap package set (see `/ov-images:debian`) because downstream layers need `gpg --dearmor` at build time.

## Usage

```yaml
# image.yml
my-image:
  layers:
    - gnupg
```

Typically used as part of the `agent-forwarding` composition layer rather than directly.

## Runtime Behavior

Provides `gpg`, `gpgconf`, `gpg-agent`, `gpg-connect-agent` binaries inside the container. When combined with SSH/GPG agent forwarding (`ov shell`, `ov start` direct mode), the container's GPG uses the host's agent for private key operations (signing, decryption) via a forwarded socket.

The container has its own keyring (public keys must be imported separately with `gpg --import`). No host keyring is mounted — only the agent socket is forwarded.

## Used In Images

Part of `agent-forwarding` composition layer, used in 27 application images including: `arch-ov`, `fedora-ov`, `nvidia`, `jupyter`, `ollama`, `openclaw`, `immich`, `comfyui`, `selkies-desktop`, and all other non-base images.

## Related Layers

- `/ov-layers:agent-forwarding` -- metalayer that includes gnupg + direnv + ssh-client
- `/ov-layers:direnv` -- environment variable loading from .envrc/.secrets
- `/ov-layers:ssh-client` -- OpenSSH client for SSH agent forwarding

## Cross-References

- `/ov:secrets` -- `ov secrets gpg` commands: key management (`import-key`, `export-key`), GPG agent setup (`setup`), health check (`doctor`), and `.secrets` file management
- `/ov:shell` -- agent socket forwarding happens at `ov shell` invocation time
- `/ov:service` -- agent forwarding in `ov start` direct mode

## When to Use This Skill

Use when the user asks about:

- GPG encryption or signing inside containers
- The `gnupg` layer
- GPG agent forwarding
- Importing public keys into container keyrings

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
