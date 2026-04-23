---
name: config
description: |
  MUST be invoked before any work involving: ov config commands, image deployment setup, quadlet generation, secrets provisioning, encrypted volumes, data seeding, or volume backing configuration.
---

# ov config -- Image Deployment Configuration

## Overview

`ov config` configures an image for deployment: generates a systemd quadlet unit, provisions container secrets, initializes encrypted volumes, and seeds data from data layers into the image's volumes (both bind mounts and podman named volumes). Requires `run_mode=quadlet`.

This is the **single entry point** for **container** deployment setup. `ov start` requires `ov config` to have been run first in quadlet mode.

**Relationship to `ov deploy add`** — `ov config` remains the primary way to create/update a quadlet and provision secrets/volumes/sidecars for a container deploy. `ov deploy add <name> <ref>` (container target) wraps both `ov config` and `ov start` and additionally handles `--add-layer` overlay synthesis (an overlay Containerfile is built before the quadlet references the resulting overlay image). `ov deploy add host` bypasses `ov config` entirely — the host target has no quadlet; it writes systemd units directly (when `--with-services` is enabled) and records every action in the ledger at `~/.config/overthink/installed/`. See `/ov:deploy` for the command family and `/ov:host-deploy` for host-target semantics.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Full setup | `ov config <image>` | Quadlet + secrets + volumes + data seed |
| Instance setup | `ov config <image> -i <instance>` | Deploy named instance with separate config |
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
| `--env` | `-e` | | Set container env var (KEY=VALUE), merged with existing vars (upsert by key) |
| `--clean` | `-c` | | Replace all env vars instead of merging (clean slate) |
| `--env-file` | | | Load env vars from file |
| `--instance` | `-i` | | Instance name (multiple containers of same image) |
| `--port` | `-p` | | Remap host port (newHost:containerPort) |
| `--keep-mounted` | | | Keep encrypted volumes mounted after setup |
| `--password` | | `auto` | `auto`: generate secrets, `manual`: prompt for each |
| `--volume` | `-v` | | Configure volume backing (name:type[:path]). Type: volume\|bind\|encrypted |
| `--bind` | | | Shorthand: configure volume as bind mount (name or name=path) |
| `--encrypt` | | | Shorthand: configure volume as encrypted (gocryptfs) |
| `--seed` | | `true` | Seed image volumes (bind mounts AND named volumes) with data from the image's data layers |
| `--no-seed` | | | Disable data seeding |
| `--force-seed` | | | Re-seed even if target directory is not empty |
| `--data-from` | | | Seed data from a different data image |
| `--no-autodetect` | | | Disable GPU/device auto-detection (skips DRINODE, DRI_NODE, HSA_OVERRIDE_GFX_VERSION injection) |
| `--ssh-key` | | `auto` | SSH public key: `auto` (default ~/.ssh key), path to .pub file, `generate`, or `none` |
| `--update-all` | | | Regenerate quadlets for all other deployed images to pick up service env changes |
| `--memory-max` | | | Cgroup `memory.max` hard OOM limit (e.g. `6g`, `500m`). Persists to deploy.yml. |
| `--memory-high` | | | Cgroup `memory.high` soft limit — reclaim pressure kicks in before OOM. Persists to deploy.yml. |
| `--memory-swap-max` | | | Cgroup `memory.swap.max` ceiling (e.g. `2g`). Persists to deploy.yml. |
| `--cpus` | | | CPU quota in cores (e.g. `2.5` for 2.5 cores). Persists to deploy.yml. |

## How It Works

1. Requires `run_mode=quadlet` (errors if set to `direct`)
2. Resolves image reference (local or remote `github.com/...`)
3. Ensures image exists in run engine (transfers if needed)
4. Extracts metadata from OCI image labels
5. Generates quadlet `.container` file in `~/.config/containers/systemd/`
6. Provisions container secrets (from `org.overthinkos.secrets` label)
7. Resolves volume backing (named, bind, or encrypted)
8. Initializes encrypted volumes (gocryptfs) if configured
9. Seeds data layers into the image's volumes (bind mounts AND podman named volumes)
10. Runs `systemctl --user daemon-reload`
11. Injects `env_provides` entries from image labels into deploy.yml `provides.env:` (resolves `{{.ContainerName}}` templates)
12. Injects `mcp_provides` entries from image labels into deploy.yml `provides.mcp:` (resolves templates, defaults transport to `http`)
13. Synthesizes `OV_MCP_SERVERS` JSON env var for consumer containers (pod-aware: same-image entries resolve to `localhost`)
14. Warns about missing `mcp_requires` servers (same pattern as `env_requires` warnings)
15. If `--update-all`, regenerates quadlets for all other deployed images (preserving Instance, Tunnel, PodName, Sidecars, and instance-scoped volume names) and reloads systemd

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

### Bind-mounting a project checkout for `ov mcp serve`

The `ov-mcp` layer declares a `project` volume at `/workspace` (renamed from `/project` in 2026-04 for a more neutral term; the volume NAME stays `project` for a stable deployer API) and sets `env: OV_PROJECT_DIR=/workspace`. Bind-mount your overthink checkout at config time so build-mode MCP tools (`image.build`, `image.list.images`, `image.inspect`) can read `image.yml`. Alternatively, skip the bind-mount and `ov mcp serve` will auto-fall back to the upstream `overthinkos/overthink` repo (see `/ov:mcp` "Project-dir wiring"):

```bash
ov config arch-ov --bind project=/home/you/overthink
ov start arch-ov
ov test mcp call arch-ov image.list.images '{}' --name ov
# → lists images from the bind-mounted /home/you/overthink
```

The `OV_PROJECT_DIR` env var is consumed by the ov binary's global `-C` / `--dir` / `OV_PROJECT_DIR` flag (`ov/main.go` calls `os.Chdir(Dir)` before Kong dispatch). See `/ov:image` "Project directory resolution" and `/ov:mcp` "Deployment: the `ov-mcp` layer" for the full pattern.

## Secret Provisioning

Secrets declared in `layer.yml` `secrets:` field are stored as OCI label metadata. At config time:

- `--password auto` (default): generates random passwords for all secrets
- `--password manual`: prompts for each secret

**Idempotent:** existing Podman secrets are never overwritten. To re-provision: `podman secret rm <name> && ov config setup <image>`.

## Data Seeding

Data layers (layers with `data:` field) stage files into `/data/<volume>/<dest>/` at build time. At config time:

- **First config** (`--seed`, default true): copies staged data into the image's volumes (both bind mounts and named volumes)
- **Subsequent config**: skips if `data_seeded` flag is set in deploy.yml
- **`--force-seed`**: re-seeds even if directory is not empty
- **`--data-from <image>`**: seeds from a separate data image instead of the target image

### Volume-kind dispatch

Seeding runs for both volume backings, with a minor difference in how podman is invoked:

- **Bind mount** (`--bind <name>` or `--bind <name>=<path>`, or `type: bind` in deploy.yml): the seeder runs `podman run --rm -v <host-path>:/seed --userns=keep-id:uid=<UID>,gid=<GID> <image> bash -c "cp -a /data/<vol>/. /seed/"`. The `--userns=keep-id` aligns the in-container UID with the real host UID so files end up owned by the host user.
- **Named volume** (the default): the seeder runs `podman run --rm -v <ov-image-name>:/seed <image> bash -c "cp -a /data/<vol>/. /seed/"` — **without** `--userns=keep-id`. Named volumes live in the rootless subuid space; the runtime container reads them without keep-id, so the seeder must write with the same identity.

For `FROM scratch` data images (`data_image: true`), bind-mount targets use `podman create` + `podman cp` (the simpler path), and named-volume targets use `podman run --mount type=image,src=<scratch>,dst=/staging,rw=false` with a lightweight `busybox:stable` helper image as the runnable side (pulled lazily on first use).

### Emptiness check

`DataProvisionInitial` only runs when the target is empty:

- **Bind mount**: checks the per-entry subdirectory (`<bind-root>/<dest>/`) via `os.ReadDir`, so data layers with distinct `dest:` values can coexist in the same bind mount.
- **Named volume**: checks the volume root via `podman volume inspect --format {{.Mountpoint}}` followed by `os.ReadDir` of the returned path. Per-entry subdirectory checks aren't available without exec'ing into a container, so multiple entries targeting the same named volume all run on the initial seed and then transition to merge-safe `cp -an` on subsequent runs.

### Data layer ordering (bind mounts only)

For **bind-mounted** targets, data layers that target the volume **root** (`dest:` absent or empty — e.g. `notebook-templates`, which drops `getting-started.ipynb` directly into `~/workspace/`) should be listed **first** in the image's `layers:` list. A root-targeted layer ordered after subdirectory-targeted layers will see the root as non-empty (because subdirs from earlier layers exist) and skip. This is a convention, not a hard-enforced rule.

This ordering caveat does not apply to named-volume targets, where the initial seed runs against the whole volume regardless of sub-paths.

### Upgrade note

Hosts that ran ov versions prior to `fix/data-seeding-complete` had their data layers silently skipped whenever the target volume was a named volume — the common default. On the first `ov config` or `ov update` after upgrading, those previously-unseeded volumes will finally be populated with their starter content. Existing user-modified content is preserved by the emptiness check; operator action is only needed if seeding reports an error.

## Encrypted Volumes

Encrypted volumes use gocryptfs. Each volume gets `{cipher,plain}` subdirectories:

- Default path: `<encrypted_storage_path>/ov-<image>-<name>/`
- Explicit path: `--volume name:encrypt:/path` stores directly at `/path/{cipher,plain}`
- Mounted via `ExecStartPre=ov config mount` in the quadlet
- Each mount runs in a `systemd-run --scope` unit (survives container restart)
- `-allow_other` flag for rootless podman with `--userns=keep-id`

### Fast path: `ov config mount` short-circuit

When every requested encrypted volume for an image is already mounted — the
typical state on service restart, because `ov-enc-<image>-<volume>.scope`
units survive container stop/restart independently of the service's cgroup —
`ov config mount <image>` short-circuits and returns without touching the
credential store at all. Output is:

    All encrypted volumes for <image> already mounted (N/N)

This makes service restarts (`systemctl --user restart ov-<image>.service`)
resilient to temporary keyring backend problems: the only reason to query the
keyring is to get the passphrase for a fresh `gocryptfs -init` or mount, and
the short-circuit skips that entirely when no new mount is needed. A broken
`default` alias in the Secret Service — for example, KeePassXC's FdoSecrets
plugin advertising a stub collection — does NOT block restarts.

When a volume IS unmounted and needs to be remounted, the normal path runs:
`resolveEncPassphraseForMount` queries the credential store via the
iteration-capable `ssClient` (not just the Secret Service default alias), so
even then the broken-stub scenario still resolves automatically by finding
the credential in a sibling healthy collection. See `/ov:enc` for the full
iteration order, the bounded `encMountDeadline` retry behavior, and the
source classification (`env`/`keyring`/`config`/`locked`/`unavailable`/`default`).

## Resource Caps

Memory and CPU caps flow through the same `security:` block as `shm_size`.
Layers can declare defaults; image/deploy overrides replace them. CLI flags
on `ov config` persist to `deploy.yml` and take effect on the next quadlet
regeneration.

```bash
# Image-level default (all instances)
ov config selkies-desktop --memory-max=6g --memory-high=5g --memory-swap-max=2g

# Per-instance override (tighter cap for one instance)
ov config selkies-desktop -i 192.241.92.221 --memory-max=8g

# CPU quota on an unrelated image
ov config jupyter --memory-max=16g --cpus=8
```

Merge rules (see `ov/security.go`):

- **Layers → image**: smallest value wins (`minCap` / `minCpus`). A tighter
  cap is a smaller blast radius, so it's the safer default.
- **Image-level and deploy-level overrides**: full replace, identical to
  how `shm_size` is handled.

Emitted as native systemd cgroup directives (`MemoryMax=`, `MemoryHigh=`,
`MemorySwapMax=`, `CPUQuota=`) in the quadlet's `[Service]` section,
guaranteed to work on every systemd version ov targets. For the runtime
(non-quadlet) path, `SecurityArgs` also emits the equivalent
`podman run --memory / --memory-reservation / --memory-swap / --cpus` flags.

`CPUQuota` uses systemd's percentage form — `--cpus=2.5` → `CPUQuota=250%`
(one full core is `100%`).

### Gotchas

- Lowercase suffixes (`6g`) are auto-normalized to `6G`. systemd silently
  parses lowercase as `infinity` — no error, just no limit — so ov coerces
  everything to the canonical uppercase form before emitting the quadlet.
- Field-level merge in deploy means `--memory-max` won't wipe co-set fields
  like `shm_size`. Unset CLI fields fall through to layer/image defaults
  rather than being replaced with zero values.
- Resource caps merge **smallest-wins** across layers (tightest cap = smallest
  blast radius). Image-level and deploy-level overrides replace entirely.

## Port-conflict detection

When a `-p HOST:CONTAINER` flag publishes onto a host port that another listener already holds, `ov config` emits a **soft warning** (does not abort) and suggests a remap:

```
Warning: port conflicts detected:
  Port 13000 is in use
    Fix: find and stop the process using port 13000
    Or remap: ov start selkies-desktop-ov --port 13001:3000
Wrote /home/atrawog/.config/containers/systemd/ov-<image>-<instance>.container
```

The quadlet is still written with the conflicting port, so `ov start` will fail at bind time unless you either free the port, re-run `ov config` with a remap, or edit `deploy.yml`. Detection uses a local `net.Listen` probe per published port at configure time. Handy for multi-instance deployments alongside a production fleet (e.g., running `/ov-images:selkies-desktop-ov` as a test instance while `/ov-images:selkies-desktop` holds the canonical 3000/9222/9224/2222 range).

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
ov config jupyter-ml-notebook --bind workspace=~/notebooks
ov start jupyter-ml-notebook
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

### Full instance removal (important: 3-step cleanup)

`ov config remove` disables the systemd service but does NOT clean the deploy.yml entry. Running `--update-all` before cleaning deploy.yml will re-create quadlets from stale entries. Full cleanup requires:

```bash
ov config remove <image> -i <instance>     # 1. Stop & disable service
ov deploy reset <image> -i <instance>       # 2. Remove deploy.yml entry
rm ~/.config/containers/systemd/ov-<image>-<instance>.container  # 3. Delete quadlet
systemctl --user daemon-reload              # 4. Reload systemd
systemctl --user reset-failed               # 5. Clear ghost units (optional)
```

Only THEN run `ov config <image> --update-all` to propagate the clean state.

### Multi-instance proxy deployment

Deploy multiple browser instances with different HTTP proxies (e.g., for selkies-desktop):

```bash
ov config selkies-desktop -i 198.145.102.110 \
  -e HTTP_PROXY=http://198.145.102.110:5466 \
  -e HTTPS_PROXY=http://198.145.102.110:5466 \
  -e "NO_PROXY=localhost,127.0.0.1" \
  -p 3006:3000 -p 9236:9222
ov start selkies-desktop -i 198.145.102.110
```

Each instance gets unique host ports. The Chrome layer's `chrome-wrapper` translates `HTTP_PROXY`/`HTTPS_PROXY` into Chrome's `--proxy-server` flag. Verify with CDP:

```bash
ov test cdp status selkies-desktop -i 198.145.102.110
ov test cdp open selkies-desktop -i 198.145.102.110 "https://ip.me"
ov test cdp eval selkies-desktop -i 198.145.102.110 <tab-id> \
  "document.querySelector('#ip-lookup').value"
```

## Service Environment Injection

When a configured image declares `env_provides` or `mcp_provides` in its layers (stored in OCI labels), `ov config` automatically injects those entries into the `provides:` section of `deploy.yml`. This enables cross-container service discovery without manual configuration. Verify that an injected MCP endpoint is actually reachable with `ov test mcp ping <image>` — see `/ov:mcp` for the full verb surface.

```yaml
# deploy.yml after `ov config ollama && ov config jupyter`
provides:
  env:
    - name: OLLAMA_HOST
      value: http://ov-ollama:11434
      source: ollama
  mcp:
    - name: jupyter
      url: http://ov-jupyter:8888/mcp
      transport: http
      source: jupyter
images:
  ollama: { ... }
  jupyter: { ... }
```

**Self-exclusion (env):** An image's own env_provides vars are filtered out of its own environment — the image uses its own `env:` (e.g., `OLLAMA_HOST=0.0.0.0`), not the service discovery URL.

**Pod-aware (MCP):** When provider and consumer share a container, MCP URLs resolve to `localhost` instead of container hostname. No self-exclusion for MCP — same-container consumers always see their own MCP servers.

**Propagation:** Use `--update-all` to regenerate quadlets for all other deployed images so they pick up the new env vars immediately. Without `--update-all`, other images pick up the env vars on their next `ov config` or `ov update`.

**Cleanup:** `ov config remove` and `ov remove` automatically remove the service's injected vars from the global env.

See `/ov:layer` for `env_provides` field documentation.

### Instance-Aware MCP Server Naming

When deploying with `-i <instance>`, `injectMCPProvides` appends `-<instance>` to each MCP server name. This prevents naming collisions when multiple instances of the same image provide the same MCP server:

```bash
# Base deployment — MCP name: "chrome-devtools"
ov config selkies-desktop --update-all

# Instance deployment — MCP name: "chrome-devtools-31.58.9.4"
ov config selkies-desktop -i 31.58.9.4 \
  -e HTTP_PROXY=http://31.58.9.4:6077 \
  -p 3001:3000 --update-all
```

Result in `deploy.yml` `provides.mcp`:
```yaml
- name: chrome-devtools
  url: http://ov-selkies-desktop:9224/mcp
  source: selkies-desktop
- name: chrome-devtools-31.58.9.4
  url: http://ov-selkies-desktop-31.58.9.4:9224/mcp
  source: selkies-desktop/31.58.9.4
```

Consumers (e.g., hermes) see both as distinct MCP servers in `OV_MCP_SERVERS`. On re-config, stale MCP entries from the same source are automatically cleaned before new entries are injected.

Source: `ov/config_image.go` (`injectMCPProvides`).

## Hermes LLM Provider Auto-Configuration Example

The hermes layer uses `-e` env vars to auto-configure its LLM provider on first start. Pass the API key at config time:

```bash
# Ollama Cloud
ov config hermes -e OLLAMA_API_KEY=your-key

# OpenRouter
ov config hermes -e OPENROUTER_API_KEY=sk-or-xxx

# Local Ollama (OLLAMA_HOST auto-injected by ov config ollama --update-all)
ov config ollama --update-all
ov config hermes
```

All providers whose keys are present get registered simultaneously. Priority (`OLLAMA_HOST` > `OLLAMA_API_KEY` > `OPENROUTER_API_KEY`) only determines the default. Override model with `-e HERMES_MODEL=...`. See `/ov-layers:hermes` for auto-provider-config details.

## Open WebUI Auto-Configuration Example

Open WebUI uses `env_requires` for mandatory admin credentials and `env_accepts` for optional LLM providers:

```bash
# Minimal: admin credentials required (hard error if missing)
eval "$(ov secrets gpg env)"
ov config openwebui \
  -e "WEBUI_ADMIN_EMAIL=$WEBUI_ADMIN_EMAIL" \
  -e "WEBUI_ADMIN_PASSWORD=$WEBUI_ADMIN_PASSWORD" \
  -e "OPENROUTER_API_KEY=$OPENROUTER_API_KEY" \
  -e "OLLAMA_API_KEY=$OLLAMA_API_KEY" \
  --update-all

# Full workstation: Ollama + Jupyter + OpenWebUI
ov config ollama
ov config jupyter --update-all
ov config openwebui ... --update-all
```

The entrypoint auto-configures: OpenRouter + Ollama Cloud as semicolon-separated `OPENAI_API_BASE_URLS`, MCP servers from `OV_MCP_SERVERS` → `TOOL_SERVER_CONNECTIONS`, Jupyter code execution from co-deployed jupyter container. `ENABLE_OLLAMA_API` is disabled unless a local Ollama is deployed via `env_provides`. See `/ov-layers:openwebui` for details.

## env_requires Enforcement

Layers can declare `env_requires` in `layer.yml` for mandatory environment variables. `ov config` performs a hard check after resolving all env vars — if any required var is missing, it aborts before writing the quadlet and prints clear instructions:

```
Error: openwebui requires the following environment variable(s):

  WEBUI_ADMIN_EMAIL — Admin email — pass via: ov config openwebui -e WEBUI_ADMIN_EMAIL=you@example.com

Set them with -e flags, --env-file, or deploy.yml env:

  ov config openwebui -e WEBUI_ADMIN_EMAIL=...
```

Source: `ov/config_image.go` (`checkMissingEnvRequires`).

## secret_requires Enforcement

Parallel to `env_requires` but for credential-backed env vars. Layers declare `secret_requires` in `layer.yml` (see `/ov:layer`). `ov config` resolves each entry from the credential store via `ResolveCredential` and, if any required value is not stored, aborts with a clear remediation message:

```
Error: openwebui requires the following credential-backed secret(s):

  WEBUI_ADMIN_PASSWORD — Initial admin account password for Open WebUI first-run setup

Store them in the credential backend. For each entry:

  ov secrets set ov/secret WEBUI_ADMIN_PASSWORD <value>

Alternatively, pass the value once via -e; it will be auto-imported:

  ov config openwebui -e WEBUI_ADMIN_PASSWORD=...
```

Source: `ov/config_image.go` (`checkMissingSecretRequires`).

## Plaintext-to-Credential Migration Hook

When `ov config <image>` runs, it automatically migrates any existing plaintext `NAME=VAL` entry in `deploy.yml`'s `env:` list whose NAME is now declared as `secret_accepts` / `secret_requires` on the image. The sequence:

1. Scan `dc.Images[deployKey(image, instance)].Env` for names that match `meta.SecretAccepts` / `meta.SecretRequires`
2. For each match: copy the value to the credential store at the layer-declared `(service, key)` path (default `ov/secret/<NAME>`), remove the entry from `dc.Env`, mark the deploy config dirty
3. On first mutation, back up `deploy.yml` → `deploy.yml.bak.<unix-timestamp>`
4. Persist the cleaned deploy config via `SaveDeployConfig`
5. Log each migrated entry on stderr: `"Migrated plaintext OPENROUTER_API_KEY from deploy.yml to credential store (ov/api-key/openrouter)"`

Idempotent — running on a clean host is a no-op. This gives pre-upgrade hosts an automatic one-time cleanup with a rollback point preserved.

Source: `ov/config_secret_migration.go` (`MigratePlaintextEnvSecrets`).

## `-e` Auto-Import for `secret_accepts` / `secret_requires`

When a `-e NAME=VAL` flag targets an env var declared as `secret_accepts` / `secret_requires` on the image, `ov config` auto-imports the value into the credential store and strips it from `c.Env` before the plaintext env merge runs. The stderr output shows each imported key:

```
Imported WEBUI_ADMIN_PASSWORD into credential store (ov/secret/WEBUI_ADMIN_PASSWORD)
Imported OPENROUTER_API_KEY into credential store (ov/api-key/openrouter)
```

The normal secret resolution path then picks up the value from the backend on the same `ov config` invocation. First-time setup is a single command; subsequent runs don't need `-e`.

Plain `env_accepts` / `env_requires` entries are unaffected — their `-e` values continue to flow through the plaintext env merge into `deploy.yml` and the quadlet as before.

Source: `ov/config_secret_migration.go` (`scrubSecretCLIEnv`).

## Provides Filtering (env_accepts / env_requires opt-in)

`env_provides` is the **supply** side of cross-container env discovery. `env_accepts` and `env_requires` are the **demand** side. The provide-resolution pipeline in `provides.go` intersects the two sets per consumer, so env vars flow through deploy.yml's `provides:` section only to consumers that explicitly asked for them.

**Why filtering exists.** Without it, every deployed service would see every other service's `env_provides` — a chrome layer would receive tailscale sidecar `TS_*` vars, a postgres layer would receive ollama's `OLLAMA_HOST`, and the env table would quickly become noise. Explicit opt-in via `env_accepts`/`env_requires` is how we enforce the principle that services must declare the contracts they rely on.

**Resolution per variable:**

| Consumer declared | Provider deployed | Result |
|---|---|---|
| `env_requires: [X]` (no default) | yes (X in `env_provides`) | X resolved, injected via `provides:` |
| `env_requires: [X]` (no default) | **no** | **`ov config` aborts with a hard error** |
| `env_requires: [X]` with default | no | default used |
| `env_accepts: [X]` | yes | X resolved, injected |
| `env_accepts: [X]` | no | var silently omitted |
| neither accepts nor requires X | yes | **var is silently dropped** (filtering in action) |
| neither accepts nor requires X | no | nothing happens |

**Sidecar interaction.** Sidecars (e.g., the tailscale sidecar) participate in the same filtering pipeline — their `TS_*` env set is routed to the sidecar container, not auto-merged into the app container. For the app to see anything from the sidecar, it must declare `env_accepts: [<var>]` or `env_requires: [<var>]`. See `/ov:sidecar` (Environment Contract) for the pattern.

**`--update-all` effect.** When filtering rules change (e.g., a layer adds a new `env_accepts` entry), `ov config <any-image> --update-all` re-runs the resolution pipeline for every deployed image and writes updated `provides:` blocks to their quadlets. Propagation is atomic per-image.

Source: `ov/provides.go` (resolution, filtering, `{{.ContainerName}}` templating), `ov/config_image.go` (`injectEnvProvides`, `injectMCPProvides`, `checkMissingEnvRequires`).

## Sidecar Attachment

Attach a sidecar container at deploy time:

```bash
ov config --list-sidecars                    # List built-in sidecar templates
ov config <image> --sidecar tailscale        # Attach tailscale sidecar
ov config <image> --sidecar tailscale \
  -e TS_HOSTNAME=my-app \
  -e "TS_EXTRA_ARGS=--exit-node=<ip> --exit-node-allow-lan-access"
```

- `--sidecar <name>` — Attach sidecar (repeatable). Saved to deploy.yml
- CLI `-e` env vars matching sidecar template keys (e.g., `TS_*`) are auto-routed to the sidecar, not the app container
- Generates pod + sidecar + app quadlet files (3 instead of 1)

See `/ov:sidecar` for full sidecar documentation.

## Environment Variable Handling

**Merge behavior (default):** `-e` performs upsert — new vars override existing vars with the same key; existing vars not in the new set are preserved. This means `ov config setup -e HTTP_PROXY=...` adds the proxy without dropping existing vars like `SSH_AUTHORIZED_KEYS`.

**Clean behavior (`-c`):** `-c`/`--clean` replaces the entire env list in deploy.yml. Use when you want to reset env vars to exactly what's specified on the command line.

Kong `sep:"none"` on all `-e` flags means commas in values are preserved (no splitting). `NO_PROXY=localhost,127.0.0.1` works correctly as a single env var.

`normalizeNoProxy()` auto-converts semicolons to commas in `NO_PROXY`/`no_proxy` values during env resolution. Legacy semicolon values in deploy.yml are auto-healed.

**NO_PROXY enrichment:** When `HTTP_PROXY` or `HTTPS_PROXY` is present, `ov config` automatically appends all deployed container hostnames to `NO_PROXY`. This is necessary because Chrome does not support CIDR ranges in NO_PROXY (unlike curl) — without explicit hostnames, Chrome routes internal traffic like `http://ov-immich-ml:2283` through the external proxy, causing Bad Gateway errors. Applied in both the main config path and `--update-all`. Source: `ov/envfile.go` (`enrichNoProxy`), `ov/deploy.go` (`DeployedContainerNames`).

**Tunnel persistence:** `ov config setup` automatically persists tunnel config from deploy.yml back to deploy.yml via `saveDeployState`. Tunnel is a deploy-time concern — see `/ov:deploy` for tunnel configuration.

**Tunnel is deploy.yml-only:** `labels.go:238` deliberately skips parsing the `org.overthinkos.tunnel` OCI image label. Tunnel config is ONLY sourced from `deploy.yml`. New instances created with `ov config setup -i <name>` do NOT inherit tunnel config from the base image's deploy.yml entry — you must manually add `tunnel: {provider: tailscale, private: all}` to the instance's deploy.yml entry, then re-run `ov config setup` to regenerate the quadlet with `ExecStartPost=tailscale serve` commands.

Source: `ov/envfile.go` (`normalizeNoProxy`), `ov/deploy.go` (`mergeEnvVars`, `saveDeployState`), `sep:"none"` in config_image.go/shell.go/commands.go/start.go.

## Cross-References

### Prerequisites

- `/ov:pull` — Required before `ov config` can read OCI labels. Remote refs (`@github.com/...`) are rejected with a redirect to `ov image pull`. If the image isn't local, `ExtractMetadata` returns `ErrImageNotLocal` and the CLI prompts for a pull.

### Deploy-mode neighbors

- `/ov:sidecar` — Sidecar containers, pod networking, Tailscale exit nodes, Environment Contract (how sidecars participate in provides filtering)
- `/ov:start` — Requires `ov config` first in quadlet mode
- `/ov:deploy` — Deploy state file (deploy.yml), sidecar pod deployment, tunnel lifecycle, instance tunnel inheritance, resource caps persistence
- `/ov:enc` — Encrypted storage details
- `/ov:secrets` — Container secret management, `ov secrets gpg set TS_AUTHKEY`
- `/ov:settings` — Runtime settings (engine, run_mode, encrypted_storage_path)
- `/ov:service` — Service lifecycle (start, stop, status, logs)
- `/ov:layer` — Volume, secret, `env_provides` / `env_requires` / `env_accepts` declarations, security resource cap fields, `service:` blocks
- `/ov:image` — Image composition, inheritance, OCI label emission, tunnel deploy.yml-only note (`labels.go:238`)
- `/ov:doctor` — Host GPU/device detection driving `appendAutoDetectedEnv()` (DRINODE, HSA_OVERRIDE_GFX_VERSION)
- `/ov:shell` — Interactive shells share the same `appendAutoDetectedEnv()` path
- `/ov-layers:chrome` — Chrome HTTP proxy (`env_accepts`), NO_PROXY auto-enrichment, resource caps, crash-loop circuit breaker
- `/ov-layers:supervisord` — Event listener pattern that pairs with the resource caps
- `/ov-layers:nvidia`, `/ov-layers:rocm` — GPU layers that consume DRINODE auto-injection
- `/ov-layers:selkies` — Pixelflux DRINODE consumer + ScreenCapture singleton

## When to Use This Skill

**MUST be invoked** when the task involves `ov config` commands, image deployment setup, quadlet generation, sidecar attachment, secret provisioning, encrypted volumes, data seeding, or volume backing configuration. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** After build, before start. `ov image build` → `ov config` → `ov start`.

Source: `ov/config_image.go` (command structs), `ov/quadlet.go` (quadlet generation), `ov/deploy.go` (deploy state), `ov/enc.go` (encrypted volumes), `ov/secrets.go` (secret provisioning), `ov/data.go` (data seeding).

## Live-deploy verification is mandatory (see `/ov:test` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-dev:disposable`). Use `ov rebuild <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vms.yml.

**After committing the source-level fix, `ov rebuild` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
