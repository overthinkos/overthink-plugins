---
name: pull
description: |
  Fetch an image from its registry into local container storage so deploy-mode
  commands can read its OCI labels.
  MUST be invoked before any work involving: ov image pull command, the
  ErrImageNotLocal error, fetching images by short name / fully-qualified ref /
  @github.com/... remote ref, or recovering deploy-mode commands that fail
  with "image X is not available locally".
---

# ov image pull -- Fetch Image Into Local Storage

## Overview

`ov image pull` fetches an image from its registry into the local container
engine's storage so deploy-mode commands (`ov shell`, `ov start`, `ov config`,
`ov alias add`, etc.) can read its OCI labels via `ExtractMetadata`. It is a
**pull-only** operation — it does not start, configure, or restart any
service. That's `ov update`'s job (see `/ov-core:update`).

This command is the prerequisite for every deploy-mode operation on a fresh
host. Since the `ov image` refactor, deploy-mode commands no longer read
`image.yml` — they read OCI labels + `deploy.yml` only. If an image isn't
in local storage, the label read fails and the CLI surfaces a friendly
recommendation pointing here.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Pull short name | `ov image pull jupyter` | Resolves registry + tag via `image.yml` (requires project directory) |
| Pull fully-qualified ref | `ov image pull ghcr.io/overthinkos/jupyter:2026.108.56` | Pulls as-is, works from anywhere |
| Pull remote project ref | `ov image pull @github.com/org/repo/image:v1` | Downloads repo, reads its `image.yml`, pulls registry ref |
| Override tag (short name) | `ov image pull jupyter --tag 2026.108.1` | Pull a specific CalVer tag |
| Override platform | `ov image pull jupyter --platform linux/arm64` | Pull a specific platform |

## Three Input Forms

### 1. Short name (requires project directory)

```bash
cd ~/overthink && ov image pull jupyter
```

Resolves `<registry>/jupyter:<tag>` via `image.yml`. Equivalent to the
two-step:

```bash
ov image inspect jupyter --format registry  # ghcr.io/overthinkos
ov image inspect jupyter --format tag       # ghcr.io/overthinkos/jupyter:2026.108.56
# …then podman pull <that-ref>
```

…but packaged as a single command.

### 2. Fully-qualified ref (no project required)

```bash
ov image pull ghcr.io/overthinkos/jupyter:2026.108.56
```

Runs `podman pull <ref>` directly. Useful from any directory, including
`/tmp`. Supports any registry that the underlying engine (`podman`/`docker`)
can authenticate against.

### 3. Remote project ref (`@github.com/...`)

```bash
ov image pull @github.com/overthinkos/overthink/jupyter:2026.108.56
ov image pull @github.com/overthinkos/overthink/jupyter          # latest git tag
```

Downloads and caches the repo, reads its `image.yml`, then pulls the
registry ref declared there. This is the **only** place `@github.com/...`
refs are accepted in `ov`. Deploy-mode commands (`ov shell`, `ov start`,
`ov config`, etc.) reject them with a message pointing users here.

## Semantics

- **Idempotent.** Running `ov image pull jupyter` twice with the image
  already local is a no-op (the engine's `pull` is layer-aware).
- **No side effects on services.** Unlike `ov update`, this command does
  not restart, reconfigure, or touch any running container.
- **Prints the resolved ref** on success. One line to stderr.
- **Respects engine selection.** Uses the run engine from
  `ResolveRuntime` (`podman` by default; `docker` if configured).

## Interaction with `ErrImageNotLocal` (the sentinel pattern)

The CLI has a centralized design for "image not in local storage" errors.
Every deploy-mode command inherits the same friendly recommendation without
per-call-site code:

1. `ExtractMetadata(engine, imageRef)` in `ov/labels.go` returns
   `ErrImageNotLocal` (wrapped with the image ref) when the image is absent
   from local storage.
2. `EnsureImage(imageRef, rt)` in `ov/transfer.go` returns the same sentinel
   when the image is absent from both run and build engines.
3. `FormatCLIError` in `ov/image.go` unwraps the sentinel at the top-level
   error boundary in `main()` and renders:

```
Error: image "X" is not available locally.
       Run 'ov image pull X' to fetch it first
```

Any deploy-mode command that calls `ExtractMetadata` or `EnsureImage`
(shell, start, stop, config, deploy, update, remove, alias, vm, service,
cdp, wl, vnc, tmux, record, dbus, logs — essentially everything outside
`ov image`) automatically participates. If you're authoring a new command
that reads image metadata, just call `ExtractMetadata` and the
recommendation falls out for free.

## Interaction with `ov update`

| Aspect | `ov image pull` | `ov update` |
|---|---|---|
| Side effects | None. Pulls bytes, prints digest. | Pulls, seeds data into volumes, restarts active service. |
| Required before deploy | Yes (first time only). | No — it calls `pull` internally if needed. |
| Use when | You want labels available but aren't ready to deploy. Or just installed `ov` on a fresh host. | You have a running service and want to roll to a new image version. |

Rule of thumb: **`pull` is the prerequisite; `update` is the refresh.**
Use `ov image pull <image>` once to seed local storage; use `ov update
<image>` every time you bump the version afterward.

## Flags

- `--tag <tag>` (default: `latest`) — Image tag when resolving a short
  name. Ignored for fully-qualified or remote-project refs (the tag is
  embedded in the ref).
- `--platform <platform>` — Target platform (default: host). Passed to
  `podman pull --platform`.

## Typical Workflows

### Fresh host, new user

```bash
# Install ov (see README Install section)
ov image pull jupyter                # pull once; labels now readable
ov config jupyter                    # generate quadlet from labels + deploy.yml
ov start jupyter                     # systemctl --user start
```

### Deploying a remote image

```bash
ov image pull @github.com/overthinkos/overthink/hermes:latest
ov config hermes
ov start hermes
```

### Fixing a broken deploy-mode command

```bash
ov shell jupyter
# Error: image "jupyter:latest" is not available locally.
#        Run 'ov image pull jupyter:latest' to fetch it first
ov image pull jupyter:latest          # follow the recommendation
ov shell jupyter                      # now works
```

## Why this command exists

Before the `ov image` refactor, deploy-mode commands had dual-mode logic:
try `image.yml` first, fall back to OCI labels. That produced drift (the
two paths could diverge) and silently pulled-and-built remote refs as a
side effect of what looked like a simple `ov shell @github.com/...` call.

The refactor drew a hard line: deploy-mode commands read labels only;
build-mode commands (`ov image …`) read `image.yml` only. `ov image pull`
is the bridge — it takes a build-mode identifier (short name or remote
repo ref) and produces a deploy-mode-consumable artifact (labels in local
storage).

## Project directory override

`ov image pull` resolves `image.yml` via `os.Getwd()` when given a short name (to resolve registry + tag). Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. Fully-qualified refs and `@github.com/...` remote refs don't need a project dir. See `/ov-build:image` "Project directory resolution".

## Cross-References

- `/ov-build:image` — family overview; `ov image pull` is one of 8 subcommands.
- `/ov-build:build` — pulls and builds are orthogonal; build creates images,
  pull fetches existing ones.
- `/ov-core:update` — rolls deployed services to a new image version
  (pulls + data-seeds + restarts).
- `/ov-build:inspect` — print resolved ref from `image.yml` without pulling.
- `/ov-core:shell`, `/ov-core:start`, `/ov-core:config`, `/ov-advanced:alias`, `/ov-advanced:vm` —
  deploy-mode commands that require a pulled image.
- `/ov-core:deploy` — deploy.yml overlay semantics applied on top of the
  labels `pull` materializes.
- `/ov-dev:go` — `ErrImageNotLocal` / `EnsureImage` / `ExtractMetadata`
  source locations (`ov/labels.go`, `ov/transfer.go`, `ov/image.go`).
