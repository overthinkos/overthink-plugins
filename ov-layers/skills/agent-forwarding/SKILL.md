---
name: agent-forwarding
description: |
  Agent forwarding support -- GPG, SSH, and direnv for .secrets workflow.
  Use when working with agent forwarding, SSH/GPG socket forwarding, or the agent-forwarding layer.
---

# agent-forwarding -- SSH/GPG Agent Forwarding Support

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` (metalayer) |
| Composition | `gnupg`, `direnv`, `ssh-client` |

## Usage

```yaml
# image.yml
my-image:
  layers:
    - agent-forwarding
```

Included in all application images (27 total). Not included in base images (`fedora`, `archlinux`) or builder images (`fedora-builder`, `archlinux-builder`).

## How Agent Forwarding Works

Agent forwarding is a **runtime** feature — sockets are bind-mounted from the host into the container at `ov shell` / `ov start` (direct mode) invocation time. This layer provides the container-side binaries (`gpg`, `ssh`, `direnv`) needed to use the forwarded sockets.

### SSH Agent Forwarding

```
Host: $SSH_AUTH_SOCK (any path)
  → Container: /run/host-ssh-auth.sock
  + SSH_AUTH_SOCK=/run/host-ssh-auth.sock (env var)
```

The container's `ssh` and `git` commands automatically use the host's SSH keys. No SSH agent runs inside the container.

**Verification:** `ov shell <image> -c 'ssh-add -l'` lists host SSH keys.

### GPG Agent Forwarding

```
Host: S.gpg-agent (detected via gpgconf --list-dirs agent-socket)
  → Container: $HOME/.gnupg/S.gpg-agent (HOME from org.overthinkos.home label)
```

The container's `gpg` finds the forwarded socket at its standard path. Private key operations (signing, decryption) go through the host's agent. No GPG agent or keyboxd runs inside the container.

**The container has its own keyring.** Public keys must be imported separately:

```bash
# Export from host, import into container
gpg --export --armor KEY_ID | ov shell <image> -c 'gpg --import'
```

**Verification:** `ov shell <image> -c 'gpg-connect-agent --no-autostart /bye'` connects to the host agent (shows "restricted mode" — this is normal for forwarded agent sockets).

### Why NOT Quadlet

Agent socket paths are **session-bound** — they live under `$XDG_RUNTIME_DIR` and change between SSH sessions, reboots, and users. Quadlet `.container` files are static systemd units generated once. Baking socket paths into quadlets would cause boot failures on headless servers managed via SSH.

**Agent forwarding is intentionally excluded from quadlet mode.** Use `ov shell` or `ov start` (direct mode) for agent access.

## Settings

| Setting | Default | Env Var | Description |
|---------|---------|---------|-------------|
| `forward_gpg_agent` | `true` | `OV_FORWARD_GPG_AGENT` | Forward host GPG agent socket |
| `forward_ssh_agent` | `true` | `OV_FORWARD_SSH_AGENT` | Forward host SSH agent socket |

```bash
ov settings set forward_gpg_agent false    # Disable GPG forwarding globally
ov settings set forward_ssh_agent false    # Disable SSH forwarding globally
ov settings reset forward_gpg_agent        # Re-enable (back to default: true)
```

### Per-Image Override (deploy.yml)

```yaml
# ~/.config/ov/deploy.yml
images:
  immich:
    forward_gpg_agent: false    # No GPG needed for photo management
    forward_ssh_agent: false    # Security: no host SSH access
  fedora-ov:
    forward_gpg_agent: true     # Explicit (same as default)
```

Resolution chain: deploy.yml per-image > global setting > default (true).

## Where Forwarding is Applied

| Command | Volumes (mounts) | Env vars | Notes |
|---------|-------------------|----------|-------|
| `ov shell` (new container) | Yes | Yes | Full forwarding |
| `ov shell` (exec into running) | No | Yes | Env only; sockets from start time |
| `ov start` (direct mode) | Yes | Yes | Full forwarding |
| `ov cmd` | No | Yes | Exec into running container |
| `ov config` / quadlet | No | No | Intentionally excluded |

## Used In Images

arch-ov, arch-test, aurora, bazzite-ai, comfyui, fedora-ov, fedora-test, githubrunner, immich, immich-ml, jupyter, nvidia, ollama, openclaw, openclaw-browser-bootc, openclaw-full, openclaw-full-ml, openclaw-full-sway, openclaw-ollama, openclaw-ollama-sway-browser, openclaw-sway-browser, python-ml, selkies-desktop, selkies-desktop-nvidia, sway-browser-vnc, unsloth-studio, valkey-test

## Related Layers

- `/ov-layers:gnupg` -- GnuPG encryption tools (part of this metalayer)
- `/ov-layers:direnv` -- direnv environment loader (part of this metalayer)
- `/ov-layers:ssh-client` -- OpenSSH client (part of this metalayer)
- `/ov-layers:sshd` -- SSH server (separate, for images that need inbound SSH)
- `/ov-layers:gocryptfs` -- encrypted filesystem (separate credential system)

## Cross-References

- `/ov:secrets` -- `ov secrets gpg` for managing `.secrets` files, GPG key management (`import-key`, `export-key`, `setup`, `doctor`), KeePassXC integration; credential store for container secrets
- `/ov:shell` -- where SSH/GPG agent forwarding happens at runtime
- `/ov:service` -- `ov start` direct mode forwarding
- `/ov:config` -- `forward_gpg_agent`, `forward_ssh_agent` settings
- `/ov:deploy` -- per-image forwarding overrides in deploy.yml

## Source

`ov/agent_forward.go` (socket detection, mount resolution), `ov/runtime_config.go` (settings).

## When to Use This Skill

**MUST be invoked** when the task involves SSH or GPG agent forwarding, the `.secrets` + direnv workflow inside containers, or the `agent-forwarding` layer. Invoke this skill BEFORE reading source code.
