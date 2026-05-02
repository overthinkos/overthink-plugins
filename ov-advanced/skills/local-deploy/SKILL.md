---
name: local-deploy
description: |
  MUST be invoked before any work involving: `target: local` deployments (was `target: host`), the Ansible-style `host:` destination field (literal `local` for direct shell, anything else routes through ssh(1) reading `~/.ssh/config` + ssh-agent), the `local:` template reference, the `user:` and `ssh_args:` Ansible-shaped overrides, the managed `~/.config/ov/ssh_config` fragment, the install ledger at `~/.config/overthink/installed/`, ReverseOp teardown, or the `--with-services`/`--allow-repo-changes`/`--allow-root-tasks` gates.
---

# Local Deploy — Applying Layers Directly to a Linux Filesystem

## Overview

`target: local` deployments apply an image or layer's install recipe directly to a Linux filesystem instead of baking it into a container image. The destination is named by the deployment's `host:` field (Ansible-style):

- `host: local` (literal) or absent → `ShellExecutor` (run on this machine).
- Anything else → `SSHExecutor` (ssh(1) reads `~/.ssh/config` + ssh-agent for keys, host-key checking, options).

The same `InstallPlan` IR that drives `ov image build` (via OCITarget) and container deploys (via PodDeployTarget) is consumed by `LocalDeployTarget`, which translates each IR step into shell commands, `podman run <builder>` invocations for compile-needing work, and systemd unit writes.

Use cases:
- Installing a focused tool set (ripgrep + uv + direnv) on your workstation without a container.
- Iterating on a layer locally, then baking it into an image.
- Pushing a profile to a remote machine over SSH (CI runner, lab box, bastion) without ad-hoc shell scripting.

## SSH config + agent are the configuration

ov contains **zero** custom SSH-key resolution. We do not read `~/.ssh/config`, we do not detect ssh-agent, we do not prompt for keys. `ssh(1)` does it all. Configure your destinations via `~/.ssh/config` `Host` stanzas, load keys into `ssh-agent`, and `ov` shells out to `ssh` with no `-i` / `-o StrictHostKeyChecking=` / `-o UserKnownHostsFile=` overrides.

For VM destinations, `ov vm create <name>` writes a managed Host stanza into `~/.config/ov/ssh_config` (one per VM, fenced with `# overthink:begin` markers) and ensures your `~/.ssh/config` has `Include ~/.config/ov/ssh_config` (also managed). After that, `ssh ov-<vmname>` works from any terminal — and `LocalDeployTarget` constructs `&SSHExecutor{Host: "ov-<vmname>"}` with no User/Port/Key.

## Quick Reference

| Action | Command |
|---|---|
| Direct local | `ov deploy add my-laptop` (`host: local` is the default) |
| SSH to remote | `ov deploy add ci-3` with `host: user@ci-3.lan` in deploy.yml |
| Reference a template | `local: dev-workstation` on the deployment |
| Tear down | `ov deploy del <name>` |
| Tear down, keep repo changes | `ov deploy del <name> --keep-repo-changes` |

## `host:` destination semantics

Reserved literal: `local`. Anything else (including `localhost`, `127.0.0.1`) goes through SSH.

```yaml
deployment:
  # Direct local — host: omitted == "local".
  my-laptop:
    target: local
    local: dev-workstation
    add_layers: [sshkeys]

  # Explicit local sentinel.
  my-laptop-explicit:
    target: local
    local: dev-workstation
    host: local

  # SSH to remote machine (ssh-config + agent supply credentials).
  ci-runner-3:
    target: local
    local: ci-runner
    host: ubuntu@ci-runner-3.lan

  # SSH with explicit port.
  bastion:
    target: local
    local: dev-workstation
    host: admin@bastion.example.com:2222

  # SSH to loopback for testing the SSH path.
  ssh-self-test:
    target: local
    local: dev-workstation
    host: localhost
```

## `user:` and `ssh_args:` — Ansible-style overrides

Two pass-through fields mirror Ansible's per-host overrides:

```yaml
# Explicit user override (Ansible's ansible_user). Used when host: has
# no "@" prefix. Cleaner than embedding the user in host: when the
# destination is an ssh-config alias.
workshop-laptop:
  target: local
  local: dev-workstation
  host: workshop-laptop.lan
  user: alice

# ssh_args: passes options through to ssh(1) (Ansible's
# ansible_ssh_extra_args). Use sparingly — ssh-config Host stanzas are
# the right home for persistent options.
via-bastion:
  target: local
  local: dev-workstation
  host: target.internal
  user: ops
  ssh_args:
    - "-o"
    - "ProxyJump=ops@bastion.example.com"
```

**Precedence rule** for `user:` vs the inline `host: <user>@<machine>` form: the inline user wins (more-specific beats more-general). When both are set with different values, the validator emits an error — pick one. When both are absent, ssh(1) reads the `User` directive from `~/.ssh/config` or falls back to `$USER`.

## Setup: one-time `Include` line

The first `ov vm create` writes:
- A Host stanza into `~/.config/ov/ssh_config` (managed block, fenced).
- An `Include ~/.config/ov/ssh_config` line into your `~/.ssh/config` (also fenced).

If you prefer to manage your `~/.ssh/config` manually, add the Include line yourself before creating any VMs. `ov vm destroy` removes the matching stanza and, when the fragment is empty, removes the Include line too.

## Passwordless sudo on remote SSH targets

Remote `target: local` deploys assume **passwordless sudo** on the destination. We do not bridge interactive sudo prompts through the SSH session. Either: (a) configure `NOPASSWD` for your user on the remote, or (b) restrict the deploy to layers that don't require root (omit `--with-services`, omit packages that need `sudo dnf`).

## Gates (opt-in flags)

| Flag | Guards |
|---|---|
| `--with-services` | systemd unit writes; `systemctl enable --now` |
| `--allow-repo-changes` | `/etc/yum.repos.d/`, `/etc/apt/`, `/etc/pacman.conf` mutations |
| `--allow-root-tasks` | arbitrary `cmd: user: root` task bodies (opaque shell) |
| `--skip-incompatible` | skip layers without a destination-matching format section |
| `--builder-image <ref>` | override the compile builder image |
| `--yes` / `-y` | implies all three gates + skips sudo preflight |

## Validation

`ov image validate` checks every `target: local` deployment:

- `local: <name>` references a `kind: local` template that exists.
- `host:` field, when non-`local`, parses via `ParseSSHTarget`.
- `user:` and `ssh_args:` only meaningful when `host:` is non-`local` (otherwise error).
- `user:` redundancy: when `host: <inline>@<machine>` and `user:` are both set with different values, error.

## Cross-References

- `/ov-build:local-spec` — author-facing reference for `kind: local` templates.
- `/ov-dev:local-infra` — Go file map for the executor/ledger surface.
- `/ov-core:deploy` — parent command family.
- `/ov-build:layer` — `services:` schema rendered as systemd units on local-target deploys.
- `/ov-advanced:vm` — managed ssh-config fragment writen on `ov vm create`.
- `/ov-build:eval` — `--verify` re-runs layer `tests:` against the deploy post-install.
- `/ov-dev:install-plan` — shared IR consumed by `LocalDeployTarget`.

## When to Use This Skill

**MUST be invoked** when the task involves `target: local` (or legacy `target: host`), the Ansible-style `host:` field, the `user:`/`ssh_args:` overrides, the managed ssh-config fragment, the install ledger, or ReverseOp teardown. Invoke this skill BEFORE reading Go source or launching Explore agents.
