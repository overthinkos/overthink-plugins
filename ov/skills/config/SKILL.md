---
name: config
description: |
  MUST be invoked before any work involving: ov config commands, image deployment setup, quadlet generation, secrets provisioning, encrypted volumes, data seeding, or volume backing configuration.
---

# ov config -- Image Deployment Configuration

## Overview

`ov config` configures an image for deployment: generates a systemd quadlet unit, provisions container secrets, initializes encrypted volumes, and seeds data into bind-backed volumes. Requires `run_mode=quadlet`.

This is the **single entry point** for deployment setup. `ov start` requires `ov config` to have been run first in quadlet mode.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Full setup | `ov config <image>` | Quadlet + secrets + volumes + data seed |
| With bind mount | `ov config <image> --bind workspace=~/project` | Bind workspace to host dir |
| With encryption | `ov config <image> --encrypt data` | Encrypted volume via gocryptfs |
| Volume config | `ov config <image> -v data:bind:/mnt/nas` | Explicit volume backing |
| Manual passwords | `ov config <image> --password manual` | Prompt for each secret |
| Re-seed data | `ov config <image> --force-seed` | Overwrite existing data |
| Seed from other image | `ov config <image> --data-from data-image:v1` | Use separate data image |
| No data seed | `ov config <image> --no-seed` | Skip data provisioning |
| Keep mounts | `ov config <image> --keep-mounted` | Keep encrypted volumes mounted after setup |
| Show status | `ov config status <image>` | Show encrypted volume status |
| Mount volumes | `ov config mount <image>` | Mount encrypted volumes |
| Unmount volumes | `ov config unmount <image>` | Unmount encrypted volumes |
| Change password | `ov config passwd <image>` | Change gocryptfs password |
| Remove config | `ov config remove <image>` | Remove quadlet and disable service |

## Subcommands

| Subcommand | Description |
|------------|-------------|
| `setup` (default) | Full setup: quadlet + secrets + encrypted volumes + data seed |
| `status` | Show encrypted volume mount status |
| `mount` | Mount encrypted volumes |
| `unmount` | Unmount encrypted volumes |
| `passwd` | Change gocryptfs password |
| `remove` | Remove quadlet file and disable systemd service |

## Setup Flags

| Flag | Short | Default | Description |
|------|-------|---------|-------------|
| `--tag` | | `latest` | Image tag to use |
| `--build` | | | Force local build instead of pulling |
| `--env` | `-e` | | Set container env var (KEY=VALUE) |
| `--env-file` | | | Load env vars from file |
| `--instance` | `-i` | | Instance name (multiple containers of same image) |
| `--port` | `-p` | | Remap host port (newHost:containerPort) |
| `--keep-mounted` | | | Keep encrypted volumes mounted after setup |
| `--password` | | `auto` | `auto`: generate secrets, `manual`: prompt for each |
| `--volume` | `-v` | | Configure volume backing (name:type[:path]). Type: volume\|bind\|encrypted |
| `--bind` | | | Shorthand: configure volume as bind mount (name or name=path) |
| `--encrypt` | | | Shorthand: configure volume as encrypted (gocryptfs) |
| `--seed` | | `true` | Seed bind-backed volumes with data from image |
| `--no-seed` | | | Disable data seeding |
| `--force-seed` | | | Re-seed even if target directory is not empty |
| `--data-from` | | | Seed data from a different data image |
| `--no-autodetect` | | | Disable GPU/device auto-detection |
| `--update-all` | | | Regenerate quadlets for all other deployed images to pick up service env changes |

## How It Works

1. Requires `run_mode=quadlet` (errors if set to `direct`)
2. Resolves image reference (local or remote `github.com/...`)
3. Ensures image exists in run engine (transfers if needed)
4. Extracts metadata from OCI image labels
5. Generates quadlet `.container` file in `~/.config/containers/systemd/`
6. Provisions container secrets (from `org.overthinkos.secrets` label)
7. Resolves volume backing (named, bind, or encrypted)
8. Initializes encrypted volumes (gocryptfs) if configured
9. Seeds data layers into bind-backed volumes
10. Runs `systemctl --user daemon-reload`
11. Injects `env_provides` entries from image labels into deploy.yml `provides.env:` (resolves `{{.ContainerName}}` templates)
12. Injects `mcp_provides` entries from image labels into deploy.yml `provides.mcp:` (resolves templates, defaults transport to `http`)
13. Synthesizes `OV_MCP_SERVERS` JSON env var for consumer containers (pod-aware: same-image entries resolve to `localhost`)
14. Warns about missing `mcp_requires` servers (same pattern as `env_requires` warnings)
15. If `--update-all`, regenerates quadlets for all other deployed images and reloads systemd

## Volume Backing

Volumes declared in layer.yml default to named volumes. At `ov config` time, backing can be changed per-volume:

```bash
# Named volume (default)
ov config my-app

# Bind mount to host directory
ov config my-app --bind workspace=~/project
ov config my-app -v workspace:bind:~/project    # Equivalent

# Encrypted (gocryptfs)
ov config my-app --encrypt data
ov config my-app -v data:encrypted              # Equivalent
ov config my-app -v data:encrypt:/mnt/ssd       # With explicit path

# Multiple volumes
ov config my-app --bind workspace=~/project --encrypt data -v models:bind
```

Auto-path for bind without explicit host path: `<volumes_path>/<image>/<name>` (default: `~/.local/share/ov/volumes/`).

## Secret Provisioning

Secrets declared in `layer.yml` `secrets:` field are stored as OCI label metadata. At config time:

- `--password auto` (default): generates random passwords for all secrets
- `--password manual`: prompts for each secret

**Idempotent:** existing Podman secrets are never overwritten. To re-provision: `podman secret rm <name> && ov config setup <image>`.

## Data Seeding

Data layers (layers with `data:` field) stage files into `/data/<volume>/<dest>/` at build time. At config time:

- **First config** (`--seed`, default true): copies staged data into bind-backed volumes
- **Subsequent config**: skips if `data_seeded` flag is set in deploy.yml
- **`--force-seed`**: re-seeds even if directory is not empty
- **`--data-from <image>`**: seeds from a separate data image instead of the target image

Only applies to bind-backed volumes (not named volumes or encrypted volumes without bind).

## Encrypted Volumes

Encrypted volumes use gocryptfs. Each volume gets `{cipher,plain}` subdirectories:

- Default path: `<encrypted_storage_path>/ov-<image>-<name>/`
- Explicit path: `--volume name:encrypt:/path` stores directly at `/path/{cipher,plain}`
- Mounted via `ExecStartPre=ov config mount` in the quadlet
- Each mount runs in a `systemd-run --scope` unit (survives container restart)
- `-allow_other` flag for rootless podman with `--userns=keep-id`

## Deploy State

All configuration is persisted to `~/.config/ov/deploy.yml`:

```yaml
images:
  my-app:
    tag: latest
    volumes:
      - name: workspace
        type: bind
        host: ~/project
    data_seeded: true
    data_source: "my-app:latest"
```

## Common Workflows

### Deploy a Jupyter notebook server

```bash
ov config jupyter-colab-ml-notebook --bind workspace=~/notebooks
ov start jupyter-colab-ml-notebook
# Open http://localhost:8888
```

### Deploy with encrypted model cache

```bash
ov config ollama --encrypt models
# Prompted for gocryptfs password on first setup
ov start ollama
```

### Remove and reconfigure

```bash
ov config remove my-app
ov config my-app --bind workspace=/new/path
```

## Service Environment Injection

When a configured image declares `env_provides` or `mcp_provides` in its layers (stored in OCI labels), `ov config` automatically injects those entries into the `provides:` section of `deploy.yml`. This enables cross-container service discovery without manual configuration.

```yaml
# deploy.yml after `ov config ollama && ov config jupyter-colab`
provides:
  env:
    - name: OLLAMA_HOST
      value: http://ov-ollama:11434
      source: ollama
  mcp:
    - name: jupyter-colab
      url: http://ov-jupyter-colab:8888/mcp
      transport: http
      source: jupyter-colab
images:
  ollama: { ... }
  jupyter-colab: { ... }
```

**Self-exclusion (env):** An image's own env_provides vars are filtered out of its own environment — the image uses its own `env:` (e.g., `OLLAMA_HOST=0.0.0.0`), not the service discovery URL.

**Pod-aware (MCP):** When provider and consumer share a container, MCP URLs resolve to `localhost` instead of container hostname. No self-exclusion for MCP — same-container consumers always see their own MCP servers.

**Propagation:** Use `--update-all` to regenerate quadlets for all other deployed images so they pick up the new env vars immediately. Without `--update-all`, other images pick up the env vars on their next `ov config` or `ov update`.

**Cleanup:** `ov config remove` and `ov remove` automatically remove the service's injected vars from the global env.

See `/ov:layer` for `env_provides` field documentation.

## Hermes LLM Provider Auto-Configuration Example

The hermes layer uses `-e` env vars to auto-configure its LLM provider on first start. Pass the API key at config time:

```bash
# Ollama Cloud
ov config hermes-full -e OLLAMA_API_KEY=your-key

# OpenRouter
ov config hermes -e OPENROUTER_API_KEY=sk-or-xxx

# Local Ollama (OLLAMA_HOST auto-injected by ov config ollama --update-all)
ov config ollama --update-all
ov config hermes
```

All providers whose keys are present get registered simultaneously. Priority (`OLLAMA_HOST` > `OLLAMA_API_KEY` > `OPENROUTER_API_KEY`) only determines the default. Override model with `-e HERMES_MODEL=...`. See `/ov-layers:hermes` for auto-provider-config details.

## Cross-References

- `/ov:start` — Requires `ov config` first in quadlet mode
- `/ov:deploy` — Deploy state file (deploy.yml) management
- `/ov:enc` — Encrypted storage details
- `/ov:secrets` — Container secret management
- `/ov:settings` — Runtime settings (engine, run_mode, encrypted_storage_path)
- `/ov:service` — Service lifecycle (start, stop, status, logs)
- `/ov:layer` — Volume and secret declarations in layer.yml
- `/ov:image` — Image composition and inheritance
- `/ov:layer` — `env_provides` field documentation
- `/ov:deploy` — Provides configuration (env + MCP)

## When to Use This Skill

**MUST be invoked** when the task involves `ov config` commands, image deployment setup, quadlet generation, secret provisioning, encrypted volumes, data seeding, or volume backing configuration. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** After build, before start. `ov build` → `ov config` → `ov start`.

Source: `ov/config_image.go` (command structs), `ov/quadlet.go` (quadlet generation), `ov/deploy.go` (deploy state), `ov/enc.go` (encrypted volumes), `ov/secrets.go` (secret provisioning), `ov/data.go` (data seeding).
