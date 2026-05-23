---
name: ov-cachyos
description: |
  Operator CachyOS workstation profile — a kind:local template + target:local
  deploy that installs the full dev stack (30 layers) onto a CachyOS host via
  ShellExecutor. Lives in the overthinkos/cachyos submodule.
  MUST be invoked before editing or applying the ov-cachyos workstation profile.
---

# ov-cachyos

The operator's CachyOS developer-workstation profile: a `kind: local` template
(`local.ov-cachyos`) plus a `target: local` + `host: local` deploy entry
(`deploy.ov-cachyos`) that applies a kitchen-sink dev stack directly to the
current machine via `ShellExecutor` — no SSH, no VM, no container.

It lives in the **`overthinkos/cachyos`** repo (git submodule at
**`image/cachyos`**), inlined in that repo's single `overthink.yml` — both the
`kind: local` template and the `kind: deploy` entry (the lone `kind: deploy` in
that repo; every disposable test bed there is `kind: eval`). Apply it with:

```bash
ov -C image/cachyos update ov-cachyos
# or, anywhere:
ov --repo overthinkos/cachyos update ov-cachyos
```

## What it installs

30 layers. Most are pulled from the main repo by **git reference**
(`@github.com/overthinkos/overthink/layers/<name>:<tag>`); the cachyos-exclusive
`ghostty`, `keepassxc-keyring`, and `wheel-nopasswd` are vendored locally in this
repo's `layers/` (resolved via its `discover:` block):

`wheel-nopasswd`, `yay`, `dev-tools`, `gh`, `pre-commit`, `tmux`, `direnv`,
`gnupg`, `keepassxc`, `keepassxc-keyring`, `tailscale`, `tailscale-up`,
`build-toolchain`, `golang`, `rust`, `nodejs`, `uv`, `claude-code`, `codex`,
`gemini`, `oracle`, `forgecode`, `devops-tools`, `docker-ce`, `kubernetes`,
`vscode`, `ghostty`, `chrome`, `nvidia`, `ov-full`.

## Key fields

| Field | Value |
|---|---|
| `install_opts.builder_image` | `ghcr.io/overthinkos/arch-builder:2026.122.2252` (OCI ref, not a layer) |
| `install_opts` | with_service, allow_repo_changes, allow_root_tasks all true |
| `env` | `EDITOR=nvim`, `PAGER=less` |
| `disposable` (deploy) | `true` — `ov rebuild ov-cachyos` is authorized |

## Deploy-scope eval probes

- `passwordless-sudo-host` — `sudo -n true` (wheel-nopasswd layer must have run)
- `nvidia-ctk-present` — `command -v nvidia-ctk`
- `nvidia-cdi-spec` — `/etc/cdi/nvidia.yaml` exists

## Composition-by-reference note

`local.ov-cachyos`'s remote layer refs are collected and materialized by the
same `CollectRemoteRefs` walk as image layer refs — the walk covers
`kind: local` template `layer:` lists, so `ov image validate` /
`ov update ov-cachyos` resolve the github-ref'd layers.

## Cross-References

- `/ov-local:local-spec` — `kind: local` template authoring reference
- `/ov-local:local-deploy` — the `target: local` deployment surface
- `/ov-distros:cachyos` — the CachyOS base of the same family
- `/ov-core:deploy` — deploy entry semantics (cross-kind name reuse: deploy.ov-cachyos ↔ local.ov-cachyos)

## When to Use This Skill

**MUST be invoked** when editing or applying the ov-cachyos workstation profile,
or reasoning about the kind:local remote-layer-ref machinery. Invoke BEFORE
reading source code or launching Explore agents.
