---
name: new
description: |
  Scaffold new layers, images, and whole projects with template files.
  MUST be invoked before any work involving: ov image new {project, image, layer} commands, creating new projects/images/layers, or scaffolding directories.
---

# ov image new -- Scaffold New Projects, Images, and Layers

The `ov image new` family groups three scaffolding verbs. See `/ov-build:image` for the full family overview and `/ov-build:layer` for the layer-authoring verb catalog it bootstraps.

## Overview

Three verbs, in decreasing scope:

| Verb | What it creates |
|---|---|
| `ov image new project <dir>` | A fresh `image.yml` (with a commented `format_config` placeholder), an empty `layers/` directory, and a `.gitignore` |
| `ov image new image <name>` | Appends an image entry to an existing `image.yml` |
| `ov image new layer <name>` | A new layer directory under `layers/<name>/` with a stub `layer.yml` |

All three are **comment-preserving**: the YAML edits route through the `yaml.v3` Node API rather than the value API, so human-authored comments and key order survive round trips. Implementation lives in `ov/scaffold_project.go` + `ov/yaml_setter.go`.

Each verb also auto-becomes an MCP tool (`image.new.project`, `image.new.image`, `image.new.layer`) via Kong reflection in `ov/mcp_server.go` — so an LLM agent driving `ov mcp serve` can scaffold a project from scratch over RPC. See `/ov-build:mcp` "Authoring tools".

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| New project | `ov image new project <dir>` | Scaffold a fresh ov project (image.yml + layers/ + .gitignore) |
| New image | `ov image new image <name> --base <ref> [--layers a,b,c]` | Append an image entry to image.yml |
| New layer | `ov image new layer <name>` | Create a new layer directory with a stub layer.yml |

## Usage

### `ov image new project <dir>`

```bash
ov image new project ~/my-project
# Creates:
#   ~/my-project/image.yml     (with commented format_config placeholder + empty images: {})
#   ~/my-project/layers/       (empty)
#   ~/my-project/.gitignore    (ignores .build/ + editor scratch files)
```

**Important caveat**: a scaffolded project is **not immediately buildable**. The `format_config:` field is commented out because `image.yml`'s `rpm:` / `deb:` / `pac:` sections need a `build.yml` to resolve. Two ways to wire one:

1. Copy the canonical upstream `build.yml` into the project and point at it:
   ```bash
   cp /path/to/overthink/build.yml ~/my-project/
   ov -C ~/my-project image set defaults.format_config build.yml
   ```
2. Reference a published release remotely (once upstream publishes a `build.yml` at the repo root on a tag):
   ```bash
   ov -C ~/my-project image set defaults.format_config '"@github.com/overthinkos/overthink/build.yml:<tag>"'
   ```

Without this step, `ov image validate` reports "must have at least one install file" because `rpm:` isn't a recognized top-level layer.yml field.

### `ov image new image <name>`

```bash
ov -C ~/my-project image new image hello \
    --base quay.io/fedora/fedora:43 \
    --layers sshd,tmux
# Appends to image.yml:
#   images:
#     hello:
#       base: quay.io/fedora/fedora:43
#       layers: [sshd, tmux]
```

Flags: `--base` (required — URL or name of another image), `--layers` (optional comma-separated layer names). Existing `image.yml` comments + key order are preserved.

### `ov image new layer <name>`

```bash
ov -C ~/my-project image new layer sshd
# Creates:
#   ~/my-project/layers/sshd/
#   ~/my-project/layers/sshd/layer.yml  (stub with empty `rpm.packages:` null)
```

Follow up with `ov layer add-rpm sshd openssh-server openssh-clients` (see `/ov-build:layer`) to populate packages without manually editing YAML. The `ov layer add-rpm` helper handles the scaffold's null `packages:` → sequence upgrade automatically.

## Workflow

The end-to-end scaffold → build flow:

1. `ov image new project ~/my-project` — create the project skeleton
2. Wire a `build.yml` (copy from overthink or reference remotely; see caveat above)
3. `ov image new layer my-svc` — create a layer
4. `ov layer add-rpm my-svc openssh-server` — populate packages (see `/ov-build:layer`)
5. `ov image new image my-app --base quay.io/fedora/fedora:43 --layers my-svc` — wire into image.yml
6. `ov image validate` — check for errors
7. `ov image build my-app` — build the image

All six steps are also callable as MCP tools (`image.new.project`, `image.new.layer`, `layer.add-rpm`, …), so an agent driving `ov mcp serve` can run this entire flow over RPC. See `/ov-build:mcp` "Authoring tools" for the worked MCP-only example.

The scaffolded `layer.yml` from step 3 is minimal (a null `rpm.packages:` list with a placeholder comment). Add sections as needed: `rpm:` / `deb:` / `pac:` / `aur:` for system packages, `env:` for runtime environment, `ports:` / `service:` / `volumes:` for services, and `tasks:` for install operations (mkdir, copy, write, download, link, setcap, cmd, build). The scaffolder does not create separate Taskfile shell scripts — all install logic flows through `tasks:` in `layer.yml`.

## Naming Rules

- Lowercase-hyphenated names only (e.g., `my-tool`, `desktop-apps`)
- Must not conflict with existing layer names

## Project directory override

`ov image new layer <name>` writes into `layers/<name>/` relative to `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `OV_PROJECT_DIR=<dir>`. See `/ov-build:image` "Project directory resolution".

## Cross-References

### `ov image` family siblings

- `/ov-build:image` -- Family overview + image.yml composition reference
- `/ov-build:build` -- Build images containing the new layer
- `/ov-build:generate` -- Containerfile generation
- `/ov-build:inspect` -- Inspect built image including the new layer
- `/ov-build:list` -- Enumerate layers (the new one shows up here)
- `/ov-build:merge` -- Post-build layer consolidation
- `/ov-build:pull` -- Pull prebuilt images (unrelated; for consumers of your new layer)
- `/ov-build:validate` -- Validate layer and image definitions

### Related skills

- `/ov-build:layer` -- Layer authoring guide, layer.yml format, install files + the `ov layer set / add-rpm / add-deb / add-pac / add-aur` editing surface
- `/ov-build:mcp` -- "Authoring tools" table + the MCP-only build-from-scratch worked example
- `/ov-dev:go` -- Implementation notes: the `yaml.v3` Node API is the reason edits preserve comments; `ov/scaffold_project.go` + `ov/yaml_setter.go` house the logic
