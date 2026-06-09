---
name: new
description: |
  Scaffold new layers, images, and whole projects with template files.
  MUST be invoked before any work involving: charly box new {project, box, candy} commands, creating new projects/images/layers, or scaffolding directories.
---

# charly box new -- Scaffold New Projects, Images, and Layers

The `charly box new` family groups three scaffolding verbs. See `/charly-image:image` for the full family overview and `/charly-image:layer` for the layer-authoring verb catalog it bootstraps.

## Overview

Three verbs, in decreasing scope:

| Verb | What it creates |
|---|---|
| `charly box new project <dir>` | A fresh `box.yml` (with a commented `format_config` placeholder), an empty `candy/` directory, and a `.gitignore` |
| `charly box new box <name>` | Appends an image entry to an existing `box.yml` |
| `charly box new candy <name>` | A new layer directory under `candy/<name>/` with a stub `candy.yml` |

All three are **comment-preserving**: the YAML edits route through the `yaml.v3` Node API rather than the value API, so human-authored comments and key order survive round trips. Implementation lives in `charly/scaffold_project.go` + `charly/yaml_setter.go`.

Each verb also auto-becomes an MCP tool (`box.new.project`, `box.new.box`, `box.new.candy`) via Kong reflection in `charly/mcp_server.go` — so an LLM agent driving `charly mcp serve` can scaffold a project from scratch over RPC. See `/charly-build:charly-mcp-cmd` "Authoring tools".

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| New project | `charly box new project <dir>` | Scaffold a fresh charly project (box.yml + candy/ + .gitignore) |
| New image | `charly box new box <name> --base <ref> [--layers a,b,c]` | Append an image entry to box.yml |
| New layer | `charly box new candy <name>` | Create a new layer directory with a stub candy.yml |

## Usage

### `charly box new project <dir>`

```bash
charly box new project ~/my-project
# Creates:
#   ~/my-project/box.yml     (with commented format_config placeholder + empty images: {})
#   ~/my-project/candy/       (empty)
#   ~/my-project/.gitignore    (ignores .build/ + editor scratch files)
```

**Important caveat**: a scaffolded project is **not immediately buildable**. The `format_config:` field is commented out because `box.yml`'s `rpm:` / `deb:` / `pac:` sections need a `build.yml` to resolve. Two ways to wire one:

1. Copy the canonical upstream `build.yml` into the project and point at it:
   ```bash
   cp /path/to/opencharly/build.yml ~/my-project/
   charly -C ~/my-project image set defaults.format_config build.yml
   ```
2. Reference a published release remotely (once upstream publishes a `build.yml` at the repo root on a tag):
   ```bash
   charly -C ~/my-project image set defaults.format_config '"@github.com/overthinkos/overthink/build.yml:<tag>"'
   ```

Without this step, `charly box validate` reports "must have at least one install file" because `rpm:` isn't a recognized top-level candy.yml field.

### `charly box new box <name>`

```bash
charly -C ~/my-project image new image hello \
    --base quay.io/fedora/fedora:43 \
    --layers sshd,tmux
# Appends to box.yml:
#   images:
#     hello:
#       base: quay.io/fedora/fedora:43
#       layers: [sshd, tmux]
```

Flags: `--base` (required — URL or name of another image), `--layers` (optional comma-separated layer names). Existing `box.yml` comments + key order are preserved.

### `charly box new candy <name>`

```bash
charly -C ~/my-project image new layer sshd
# Creates:
#   ~/my-project/candy/sshd/
#   ~/my-project/candy/sshd/candy.yml  (stub with empty `rpm.packages:` null)
```

Follow up with `charly candy add-rpm sshd openssh-server openssh-clients` (see `/charly-image:layer`) to populate packages without manually editing YAML. The `charly candy add-rpm` helper handles the scaffold's null `package:` → sequence upgrade automatically.

## Workflow

The end-to-end scaffold → build flow:

1. `charly box new project ~/my-project` — create the project skeleton
2. Wire a `build.yml` (copy from opencharly or reference remotely; see caveat above)
3. `charly box new candy my-svc` — create a layer
4. `charly candy add-rpm my-svc openssh-server` — populate packages (see `/charly-image:layer`)
5. `charly box new box my-app --base quay.io/fedora/fedora:43 --layers my-svc` — wire into box.yml
6. `charly box validate` — check for errors
7. `charly box build my-app` — build the image

All six steps are also callable as MCP tools (`box.new.project`, `box.new.candy`, `candy.add-rpm`, …), so an agent driving `charly mcp serve` can run this entire flow over RPC. See `/charly-build:charly-mcp-cmd` "Authoring tools" for the worked MCP-only example.

The scaffolded `candy.yml` from step 3 is minimal (a null `rpm.packages:` list with a placeholder comment). Add sections as needed: `rpm:` / `deb:` / `pac:` / `aur:` for system packages, `env:` for runtime environment, `port:` / `service:` / `volume:` for services, and `task:` for install operations (mkdir, copy, write, download, link, setcap, cmd, build). The scaffolder does not create separate Taskfile shell scripts — all install logic flows through `task:` in `candy.yml`.

## Naming Rules

- Lowercase-hyphenated names only (e.g., `my-tool`, `desktop-apps`)
- Must not conflict with existing layer names

## Project directory override

`charly box new candy <name>` writes into `candy/<name>/` relative to `os.Getwd()`. Override with `-C <dir>` / `--dir <dir>` / `CHARLY_PROJECT_DIR=<dir>`. See `/charly-image:image` "Project directory resolution".

## Cross-References

### `charly box` family siblings

- `/charly-image:image` -- Family overview + box.yml composition reference
- `/charly-build:build` -- Build images containing the new layer
- `/charly-build:generate` -- Containerfile generation
- `/charly-build:inspect` -- Inspect built image including the new layer
- `/charly-build:list` -- Enumerate layers (the new one shows up here)
- `/charly-build:merge` -- Post-build layer consolidation
- `/charly-build:pull` -- Pull prebuilt images (unrelated; for consumers of your new layer)
- `/charly-build:validate` -- Validate layer and image definitions

### Related skills

- `/charly-image:layer` -- Layer authoring guide, candy.yml format, install files + the `charly candy set / add-rpm / add-deb / add-pac / add-aur` editing surface
- `/charly-build:charly-mcp-cmd` -- "Authoring tools" table + the MCP-only build-from-scratch worked example
- `/charly-internals:go` -- Implementation notes: the `yaml.v3` Node API is the reason edits preserve comments; `charly/scaffold_project.go` + `charly/yaml_setter.go` house the logic
