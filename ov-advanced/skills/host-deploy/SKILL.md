---
name: host-deploy
description: |
  MUST be invoked before any work involving: `ov deploy add host` / `ov deploy del host`, applying layer recipes to the local filesystem, the host-target install ledger, ReverseOp teardown, the host-specific `--with-services`/`--allow-repo-changes`/`--allow-root-tasks` gates, sudo batching, or the `~/.config/overthink/installed/` directory.
---

# Host Deploy — Applying Layers to the Local Filesystem

## Overview

`ov deploy add host <ref>` applies an image or layer's install recipe directly to the invoking user's host filesystem instead of baking it into a container image. The same InstallPlan IR that drives `ov image build` (via OCITarget) and container deploys (via ContainerDeployTarget) is consumed by `HostDeployTarget`, which translates each IR step into shell commands, `podman run <builder>` invocations for compile-needing work, and systemd unit writes.

Use cases:
- Installing a focused tool set (ripgrep + uv + pixi env) on your workstation without standing up a container.
- Iterating on a layer by deploying it to the host, verifying behaviour, then baking it into an image.
- Overlaying team-specific config (sshkeys, vimrc) onto a base image's layer set via `--add-layer`.

Host deploys are **singletons per machine**: there is exactly one `host` deploy at any time. Container deploys coexist freely (`my-dev`, `postgres-staging`, etc.). See `/ov-core:deploy` for the command family overview and `/ov-dev:install-plan` for the IR.

## Quick Reference

| Action | Command |
|---|---|
| Apply layer to host | `ov deploy add host <ref>` |
| Apply image to host | `ov deploy add host fedora-coder` |
| Overlay extra layers | `ov deploy add host fedora-coder --add-layer <ref> --add-layer <ref>` |
| Dry-run | `ov deploy add host <ref> --dry-run [--format=json]` |
| Tear down | `ov deploy del host` |
| Tear down, keep repo changes | `ov deploy del host --keep-repo-changes` |
| Re-run layer tests post-deploy | `ov deploy add host <ref> --verify` |

## Gates (opt-in flags)

Host deploys touch global system state; the gates make intent explicit. All gates are off by default. `--yes` enables all three plus skips sudo preflight.

| Flag | Guards |
|---|---|
| `--with-services` | Writing systemd units + drop-ins; `systemctl enable --now`; `systemctl daemon-reload` |
| `--allow-repo-changes` | Repo config mutations: structured `rpm:/deb:/pac: repos:`, `rpm: copr:`, `rpm: modules:`, and root `cmd:` tasks with `dnf copr`/`dnf config-manager addrepo`/`pacman-key`/`add-apt-repository`/release-rpm install |
| `--allow-root-tasks` | Arbitrary `cmd: user: root` task bodies (opaque shell). Structured package installs and file placements are not gated (inspectable). |
| `--skip-incompatible` | Skip layers without a host-matching format section instead of failing the whole deploy |
| `--builder-image <ref>` | Override the compile-builder image (default matches host distro family) |
| `--yes` / `-y` | Implies all three gates + skips sudo preflight |

All gates are strictly host-target; `ov deploy add my-dev fedora-coder --with-services` silently ignores the flag on the container target.

## HostDeployTarget execution model

`HostDeployTarget.Emit` (`ov/deploy_target_host.go`) walks the IR, groups contiguous same-`(Scope, Venue)` steps into batches, and emits one shell block per batch:

| (Scope, Venue) | Execution form |
|---|---|
| `ScopeSystem` + `VenueHostNative` | `sudo bash <<'OV_ROOT' … OV_ROOT` |
| `ScopeUser` + `VenueHostNative` | `bash <<'OV_USER' … OV_USER` as invoking user |
| any + `VenueContainerBuilder` | `podman run <builder> bash -s < script` |
| any + `VenueSkip` | No-op with reason logged |

Per-layer flow:
1. Layer's `vars:` exported in each shell block as `export K=V`.
2. `BUILD_ARCH=$(uname -m)` and `CTX=<absolute-layer-dir>` exported (`CTX` replaces `/ctx/` in cmd bodies — 3 occurrences in the current layer set).
3. `set -e` at the top of every block.
4. Ledger entries written per-step as each step completes; mid-batch failure leaves a correct partial record.
5. Final shell-profile managed block ensured in bash/zsh/fish init file (see `/ov-dev:host-infra`).

At session start (once), `sudo -v` refreshes the sudo timestamp so subsequent `sudo bash` blocks don't re-prompt within the 5-minute cache window. `--yes` skips the preflight.

## Ledger at `~/.config/overthink/installed/`

Every host deploy writes records under the invoking user's config dir. The ledger is the source of truth for what got installed; `ov deploy del host` reads it to reverse precisely what was done.

```
~/.config/overthink/installed/
├── .lock                              # flock — serializes concurrent ov deploy sessions
├── deploys/
│   └── <deploy-id>.json               # per-deploy: image, target, layers, add_layers
└── layers/
    ├── ripgrep.json                   # per-layer: version, deployed_by, steps, reverse_ops
    └── pre-commit.json
```

Per-layer records carry a `deployed_by: [<deploy-id>, …]` set. When a deploy is torn down, its ID is removed from each layer's set; only when the set empties does the layer's actual reversal (package remove, env.d file delete, service disable, etc.) run. This is the refcount mechanism that makes overlapping deploys safe.

## The 15 ReverseOp kinds

Every step's `Reverse()` method emits a list of `ReverseOp` values recorded in the layer ledger. `ov deploy del host` executes them in LIFO order. A full catalogue:

| Kind | Effect |
|---|---|
| `package-remove` | `sudo dnf remove -y <pkgs>` / `apt-get purge -y` / `pacman -Rs --noconfirm` |
| `pixi-env-remove` | `rm -rf $HOME/.pixi/envs/<name>` |
| `cargo-uninstall` | `cargo uninstall <binary>` |
| `npm-uninstall-g` | `npm uninstall -g <package>` |
| `rm-file-system` | `sudo rm -f <path>` |
| `rm-file-user` | `rm -f <path>` (no sudo) |
| `rm-dir-recursive` | `rm -rf <path>` |
| `service-disable` | `systemctl [--user] disable --now <unit>` (skipped with `--keep-services`) |
| `service-remove` | Delete the unit file (skipped with `--keep-services`) |
| `remove-dropin` | Delete a drop-in `.conf` and, if empty, the parent `.d` dir |
| `restore-enabled` | Re-enable a packaged unit that was enabled before ov touched it |
| `remove-managed-block` | Strip the shell-profile managed block (session-level, not per-op) |
| `remove-envd-file` | Delete `~/.config/overthink/env.d/<layer>.env` |
| `remove-repo-file` | `sudo rm -f /etc/yum.repos.d/<file>.repo` (refcount-aware; skipped with `--keep-repo-changes`) |
| `copr-disable` | `sudo dnf -y copr disable <repo>` (skipped with `--keep-repo-changes`) |

See `/ov-dev:host-infra` for the executor implementations (`reverse_ops.go`).

## Container-builder invocation for host deploys

For `VenueContainerBuilder` steps (pixi, npm, cargo, aur), `HostDeployTarget` delegates to the existing multi-stage builder image via `podman run`:

```bash
podman run --rm \
  --user "$(id -u):$(id -g)" \
  -v "$HOME/.pixi":"$HOME/.pixi":rw \
  -v "$HOME/.cargo":"$HOME/.cargo":rw \
  -v "$HOME/.npm-global":"$HOME/.npm-global":rw \
  -v "$HOME/.cache/ov":"$HOME/.cache/ov":rw \
  -v "$LAYER_SRC":/work:ro \
  -e HOME="$HOME" \
  -e PIXI_CACHE_DIR="$HOME/.cache/ov/pixi" \
  -e NPM_CONFIG_PREFIX="$HOME/.npm-global" \
  -e CARGO_HOME="$HOME/.cargo" \
  -w /work \
  <builder-image> bash -s < <stage-script>
```

Properties:
- UID/GID match host user — artifacts are owned correctly.
- Only specific build-output subdirs are bind-mounted — no `$HOME/.ssh` exposure.
- `HOME=$HOME` — path-baking in shebangs resolves correctly (container and host see the same absolute paths for the mounted subdirs).
- Cache mounts shared across deploys via `$HOME/.cache/ov`.

`aur:` layers are Arch-only. On an Arch host, `archlinux-builder` produces `.pkg.tar.zst` in `/tmp/aur-pkgs/` (bind-mounted to a host staging dir), then `sudo pacman -U` installs them. On non-Arch hosts, `ov deploy add host` refuses with a clean error — format incompatibility (`.pkg.tar.zst` can't be consumed by dnf or apt).

Glibc preflight: the builder image's glibc must be ≤ the host's glibc for compiled artifacts to run. Mismatch triggers a refusal.

## Shell profile integration

Each layer's `env:` + `path_append:` materialize as a file at `~/.config/overthink/env.d/<layer>.env`. A managed block in the user's shell init sources all env.d files:

```sh
# overthink:begin (managed by ov; do not edit inside this block)
for f in "$HOME/.config/overthink/env.d"/*.env; do [ -r "$f" ] && . "$f"; done
# overthink:end
```

Target file per shell:
- **bash** — `~/.profile` (login) + `~/.bashrc` (interactive)
- **zsh** — `~/.zshenv`
- **fish** — `~/.config/fish/conf.d/overthink.fish`

Login shell detected from `$SHELL` / `getent passwd $USER`. See `/ov-dev:host-infra` for the managed-block fencing protocol.

## Examples

```bash
# Install ripgrep on the host via dnf/pacman/apt
ov deploy add host ripgrep

# Deploy a whole image's layer set to the host
ov deploy add host fedora-coder --with-services --yes

# Overlay a private layer onto a shared base image
ov deploy add host fedora-coder --add-layer ./private-overlay.yml --yes

# Remote-ref overlay
ov deploy add host fedora-coder \
  --add-layer github.com/team-acme/configs/layers/sshkeys \
  --with-services

# Dry-run — preview the InstallPlan as JSON
ov deploy add host fedora-coder --dry-run --format=json

# Tear down, keep repo-config changes (don't remove rpmfusion.repo)
ov deploy del host --keep-repo-changes

# Tear down the whole host deploy
ov deploy del host --yes
```

## Concurrency

`~/.config/overthink/installed/.lock` is taken via `flock(2)` for the full duration of each `ov deploy add host` / `ov deploy del host` session. A second concurrent invocation blocks on the lock until the first completes — prevents partial-state races.

## Cross-References

- `/ov-core:deploy` — Parent command family; deploy.yml schema; container target
- `/ov-build:layer` — Unified `services:` schema (use_packaged + custom entries) that host deploys render via systemd
- `/ov-build:build` — Build-mode Containerfile generation; three-phase template split (prepare/install/cleanup × container/host)
- `/ov-core:update` — `--pull` equivalent on `ov deploy add`
- `/ov-build:eval` — `--verify` re-runs layer `tests:` against the host post-deploy
- `/ov-dev:install-plan` — InstallPlan IR that HostDeployTarget consumes; 8 step kinds; DeployTarget interface
- `/ov-dev:host-infra` — Implementation: `hostdistro.go` (distro alias table), `install_ledger.go` (flock protocol + JSON shape), `builder_run.go` (podman argv), `shell_profile.go` (managed blocks), `reverse_ops.go` (15 executors), `service_render.go` (ServiceEntry → unit text), `deploy_ref.go` (4 ref forms)

**Canonical layer examples illustrating host-deploy behaviors:**
- `/ov-foundation:ripgrep` — Pure rpm/deb/pac system-package layer
- `/ov-coder:uv` — Download-task layer placing a single binary at `/usr/local/bin`
- `/ov-coder:pre-commit` — Multi-builder layer (pixi + npm + cargo tasks)
- `/ov-ollama:ollama` — Custom `services:` entry rendered to systemd unit
- `/ov-foundation:postgresql` — `use_packaged:` entry with drop-in overrides
- `/ov-foundation:rpmfusion` — Repo-config-mutating layer (requires `--allow-repo-changes`)
- `/ov-coder:sshd` — Layer using both `use_packaged:` (distro-shipped sshd.service) and custom service (sshd-wrapper)

## When to Use This Skill

**MUST be invoked** when the task involves `ov deploy add host`, `ov deploy del host`, the `~/.config/overthink/installed/` ledger, any of the host-target gates, or ReverseOp behaviour. Invoke this skill BEFORE reading Go source or launching Explore agents.
