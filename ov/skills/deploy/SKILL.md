---
name: deploy
description: |
  MUST be invoked before any work involving: `ov deploy add`/`ov deploy del` commands, quadlet generation, volume backing, tunnels (Tailscale/Cloudflare), `add_layers:` overlay, or per-machine deploy overlays.
---

# Deploy - Deployment Configuration

## Overview

`ov deploy` is the parent verb for applying and tearing down deployments, plus managing `deploy.yml` overrides. The command family has two distinct surfaces:

1. **Execution verbs** — `ov deploy add <name>` / `ov deploy del <name>`. Apply or reverse a deployment. The literal name `host` targets the local filesystem via `HostDeployTarget`; any other name is a container deployment managed via `ContainerDeployTarget` (overlay Containerfile + quadlet/podman). See `/ov:host-deploy` for the host-target deep dive.
2. **Config-file management** — `ov deploy show/export/import/reset/path/status`. Read and mutate `~/.config/ov/deploy.yml` itself.

`ov start` / `ov stop` remain as ergonomic wrappers: `ov start <image>` is equivalent to `ov deploy add <image> <image>` with the container target; `ov stop <name>` is `ov deploy del <name>`. New scripts should prefer the explicit `ov deploy add`/`ov deploy del` forms, especially when using `--add-layer` overlays or the `host` target.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Apply container deploy | `ov deploy add <name> <ref>` | Compile layers + build overlay if `add_layers:` present + run via quadlet |
| Apply host deploy | `ov deploy add host <ref>` | Apply layers directly to local filesystem (see `/ov:host-deploy`) |
| Tear down deploy | `ov deploy del <name>` | Stop container + reverse ReverseOps (host) + ledger cleanup |
| Dry-run | `ov deploy add <name> <ref> --dry-run [--format=json]` | Print the InstallPlan without executing |
| Layer overlay | `ov deploy add <name> <ref> --add-layer <ref>` | Extra layer(s) applied on top; repeatable |
| Configure deployment | `ov config <image>` | Generate .container file + save deploy.yml |
| Configure instance | `ov config <image> -i <instance>` | Generate instance-specific quadlet + deploy entry |
| Configure volume backing | `ov config <image> --bind name` | Set volume as host bind mount |
| Provision data | `ov config <image> --seed` | Auto-provision data layers into bind mounts (default) |
| Deploy status | `ov deploy status` | Audit deploy.yml vs quadlet sync |
| Show overrides | `ov deploy show [image]` | Display deploy.yml contents |
| Show instance overrides | `ov deploy show <image> -i <instance>` | Display instance-specific overrides |
| Import config | `ov deploy import <files>` | Merge files into deploy.yml |
| Reset config | `ov deploy reset [image]` | Remove deploy.yml overrides |
| Reset instance config | `ov deploy reset <image> -i <instance>` | Remove instance overrides |
| Push to registry | `ov image build --push` | Multi-platform push |

For service lifecycle commands (start/stop/status/logs/update/remove), see `/ov:service`. For VM deployment, see `/ov:vm`. For encrypted storage, see `/ov:enc`. For host-target semantics, see `/ov:host-deploy`. For the Go IR that drives both targets, see `/ov-dev:install-plan`.

## Command Family: `add` / `del`

### `ov deploy add <name> [<ref>]`

Applies a deployment. `<name>` selects the target:

- **`host`** (literal) — apply layers to the local filesystem via `HostDeployTarget`. One host deploy per machine (singleton). See `/ov:host-deploy`.
- **Any other string** — container deployment. Multiple container deploys coexist (`my-dev`, `postgres-staging`, etc.); each gets its own quadlet, container name, and deploy.yml entry.

`<ref>` accepts four forms, auto-detected:

| Form | Example | Resolution |
|---|---|---|
| Local image name | `fedora-coder` | Looked up in current project's `image.yml` |
| Local layer name | `pre-commit` | Looked up in current project's `layers/` directory |
| Local YAML path | `./custom.yml`, `/abs/path/layer.yml` | File's top-level keys classify image vs layer |
| Remote repo | `github.com/owner/repo[/images/<n>\|/layers/<n>][@ref]` | Fetched via existing `--repo` cache |

Disambiguation: a ref containing `/layers/` resolves to a layer; `/images/` to an image. For local names, `image.yml` is checked before `layers/`; same-named entries in both are a hard error. The legacy `@host/org/repo:version` form (used by `depends:` and `layers:` in layer.yml) is also accepted.

When `<ref>` is omitted, the ref falls back to `deploy.yml['deploys'][<name>]['image']` (or the deploy key itself if no explicit image is declared).

### `ov deploy add` flags

**Universal:**
- `--tag <calver>` — override deploy.yml tag
- `--dry-run` — print the InstallPlan without executing
- `--format table|json` — with `--dry-run`
- `--pull` — force re-fetch of remote refs / image pull
- `--verify` — run layer `tests:` post-deploy
- `--add-layer <ref>` — repeatable; extra layer(s) applied on top (all 4 ref forms)

**Host-target-specific** (silently ignored on container deploys):
- `--with-services` — opt-in for systemd unit installation (packaged-unit enable + drop-ins)
- `--allow-repo-changes` — opt-in for repo config mutations (rpmfusion, copr, external `.repo` files)
- `--allow-root-tasks` — opt-in for arbitrary `cmd: user: root` task bodies
- `--skip-incompatible` — skip layers lacking a host-matching format section instead of failing
- `--builder-image <ref>` — override the compile-builder image
- `--yes` / `-y` — all three gates plus skip sudo preflight

### `ov deploy del <name>`

Reverses a deployment. Gated host-side reversal respects `--keep-repo-changes` and `--keep-services`. Container teardown: `podman stop` + `rm` + overlay image removal (unless `--keep-image`) + ledger cleanup. See `/ov:host-deploy` for the full 15-kind ReverseOp table.

### `add_layers:` overlay mechanism

Both container and host targets accept extra layers at deploy time via `--add-layer <ref>` (repeatable) or a `deploy.yml['deploys'][<name>]['add_layers']` list. Semantics:

- **Container target**: synthesizes an overlay Containerfile (`FROM <base-image>` + the extra layers' build steps) and builds a deterministic overlay image tagged `<deploy-name>-overlay:<short-hash>`. The deploy runs the overlay, not the base image. Re-running with different overlays rebuilds. `ov deploy del <name>` removes the overlay unless `--keep-image`.
- **Host target**: the compiler merges the image's layers with `add_layers:`, topo-sorts the union, and compiles one `InstallPlan` covering the combined set. The ledger records which layers (base + overlay) were applied so teardown reverses everything.

Ref forms for `--add-layer` are identical to the primary `<ref>` positional (local name / local path / remote / legacy `@` form).

## Examples

**Deploy a local image as a container:**
```bash
ov deploy add my-dev fedora-coder
# Uses deploy.yml['deploys']['my-dev'] for volumes/ports/env/tunnel.
ov deploy del my-dev
```

**Deploy directly to the host:**
```bash
ov deploy add host fedora-coder --with-services --yes
ov deploy del host --yes
```

**Deploy from a remote repo:**
```bash
ov deploy add my-coder github.com/overthinkos/overthink/images/fedora-coder@main
ov deploy add host github.com/team-acme/private-configs/layers/my-team-tools
```

**Add overlay layers:**
```bash
ov deploy add host fedora-coder \
    --add-layer team-extras \
    --add-layer github.com/team/configs/layers/sshkeys \
    --add-layer ./private.yml \
    --with-services
```

**Dry-run to preview the plan:**
```bash
ov deploy add host fedora-coder --dry-run --format=json
```

**`ov start`/`ov stop` equivalence:**
```bash
ov start fedora-coder            # == ov deploy add fedora-coder fedora-coder (container target)
ov stop fedora-coder             # == ov deploy del fedora-coder
```

## Quadlet Generation

User-level systemd services via podman quadlet. Generated by `ov config`.

### Generated File

Path: `~/.config/containers/systemd/ov-<image>.container` (or `ov-<image>-<instance>.container` with `-i`).

Contents include:
- `[Container]` section: image reference, container name, port mappings, volumes, environment
- `[Service]` section: restart policy, lifecycle hooks
- `[Install]` section: `WantedBy=default.target` (omitted for encrypted services without keyring backend)
- `PodmanArgs=` for security settings (privileged, capabilities, devices)
- `Volume=` for named volumes and plain bind mounts
- `Environment=` / `EnvironmentFile=` for env vars
- `ExecStartPost=` / `ExecStopPost=` for tunnel commands

Service name: `ov-<image>.service`. Container name: `ov-<image>`. Entrypoint: determined by build.yml `init:` section for the configured init system. Encrypted volumes are mounted via `ExecStartPre=ov config mount` in the quadlet, which creates transient `ov-enc-<image>-<volume>.scope` units for each encrypted volume. These scope units are independent of the container service — they survive stop/restart (see `/ov:enc`). With Secret Service backend: auto-starts after login (ExecStartPre waits for keyring unlock, `TimeoutStartSec=0`). With KeePass or no backend: requires `ov start` (no `WantedBy=default.target`).

### Security in Quadlet

Layer and image-level security settings become `PodmanArgs=` in the quadlet file:

- `privileged: true` -> `PodmanArgs=--privileged`
- `cap_add` -> `PodmanArgs=--cap-add=<CAP>`
- `devices` -> `PodmanArgs=--device=<DEV>`
- `security_opt` -> `PodmanArgs=--security-opt=<OPT>`

Source: `ov/security.go`, `ov/quadlet.go`.

### Image Transfer

When `engine.build=docker`, `ov config` auto-detects if the image is missing from podman and transfers via `docker save | podman load`. `ov update` re-transfers if needed.

Source: `ov/quadlet.go` (generation), `ov/commands.go` (command structs).

## Tunnel Configuration

Expose services outside the container host via tunnels. Tunnel config lives exclusively in `deploy.yml` — it is NOT in `image.yml` or OCI image labels. `ov config setup` persists tunnel config automatically via `saveDeployState`.

### Tailscale Serve (tailnet-private, default)

Exposes a port to your Tailscale network only. No FQDN needed -- Tailscale handles TLS automatically. Any port works for tailnet-only serve.

```yaml
tunnel: tailscale
# or expanded:
tunnel:
  provider: tailscale
  private: all       # all image ports on tailnet
```

**CRITICAL: `bind_address` must be `127.0.0.1` (the default).** Setting `0.0.0.0` causes the container to bind on the Tailscale interface, preventing Tailscale from intercepting TLS. Result: HTTPS fails with `wrong version number`.

### Tailscale Funnel (public internet)

Exposes a port to the public internet via Tailscale's edge network. Funnel restricts HTTPS ports to: 443, 8443, 10000.

```yaml
tunnel:
  provider: tailscale
  public: [8080]    # funnel ports must be 443, 8443, or 10000
```

### Cloudflare Tunnel

Routes traffic through Cloudflare's network. Requires `fqdn`. Prerequisite: `cloudflared tunnel login` (one-time auth).

```yaml
tunnel:
  provider: cloudflare
  port: 3001
  tunnel: my-tunnel    # optional, defaults to ov-<image>
fqdn: "app.example.com"
```

`ov config` handles the full tunnel lifecycle automatically:
1. Creates the Cloudflare tunnel (`cloudflared tunnel create`) if it doesn't exist
2. Writes the tunnel config YAML (`~/.config/ov/tunnels/<name>.yml`) with ingress rules
3. Routes DNS with `--overwrite-dns` (creates or updates CNAME to tunnel)
4. Generates a companion systemd service (`ov-<image>-tunnel.service`)
5. Enables the tunnel service and adds `Wants=` to the container quadlet

`ov start` then starts both the container and the tunnel service together.

### Backend Schemes

Port protocols declared in `layer.yml` control the backend URL scheme used by tunnel commands. The protocol flows from layer → OCI label (`org.overthinkos.port_protos`) → tunnel command.

**Tailscale serve/funnel schemes:**

| Scheme | Target URL | Tailscale flag | Use case |
|--------|-----------|----------------|----------|
| `http` (default) | `http://127.0.0.1:PORT` | `--https` | Plain HTTP backends |
| `https` | `https://127.0.0.1:PORT` | `--https` | HTTPS with valid cert |
| `https+insecure` | `https+insecure://127.0.0.1:PORT` | `--https` | HTTPS with self-signed cert (e.g., Traefik) |
| `tcp` | `tcp://127.0.0.1:PORT` | `--tcp` | Raw TCP forwarding |
| `tls-terminated-tcp` | `tcp://127.0.0.1:PORT` | `--tls-terminated-tcp` | TLS-terminated TCP |

**Cloudflare tunnel schemes:**

| Scheme | Ingress service | Use case |
|--------|----------------|----------|
| `http` (default) | `http://localhost:PORT` | HTTP origins |
| `https` | `https://localhost:PORT` | HTTPS origins (use with `noTLSVerify`) |
| `tcp` | `tcp://localhost:PORT` | Raw TCP (requires client-side cloudflared) |
| `ssh` | `ssh://localhost:PORT` | SSH tunneling |
| `rdp` | `rdp://localhost:PORT` | RDP streaming |
| `smb` | `smb://localhost:PORT` | SMB/CIFS file sharing |

**UDP** ports are never tunneled — a warning is printed. UDP traffic works directly between tailnet nodes.

`ov image validate` checks port schemes against provider capabilities. For example, `ssh` is valid for Cloudflare but not Tailscale; `tls-terminated-tcp` is valid for Tailscale but not Cloudflare.

See `/ov:layer` for port protocol syntax in `layer.yml`.

### Multi-Port Tailscale Serve

When `private: all` or `ports: all`, every image port gets its own `tailscale serve` command with scheme-appropriate flags:

```yaml
tunnel:
  provider: tailscale
  private: all
```

- `tailscale serve --bg --https=PORT http://127.0.0.1:PORT` (http, default)
- `tailscale serve --bg --https=PORT https+insecure://127.0.0.1:PORT` (https+insecure)
- `tailscale serve --bg --tcp=PORT tcp://127.0.0.1:PORT` (tcp)
- `tailscale serve --bg --tls-terminated-tcp=PORT tcp://127.0.0.1:PORT` (tls-terminated-tcp)

Quadlet generates multiple `ExecStartPost=` and `ExecStopPost=` lines. Requires `tailscale set --operator=$USER` for non-root access.

Port protocols are stored in the `org.overthinkos.port_protos` image label so deploy-mode commands work without access to the original layer definitions. (Pre-refactor, `ov shell @github.com/...` could fetch a remote repo and build on the fly; that path was deleted. Remote refs now require `ov image pull` first — see `/ov:pull`.)

### Instance Tunnel Inheritance

**Critical:** When deploying instances with `ov config setup -i <name>`, tunnel config is NOT auto-inherited from the base image's deploy.yml entry. Each instance must have its own `tunnel:` section in deploy.yml. Without it, the generated quadlet will have no `ExecStartPost=tailscale serve` commands and the instance will be unreachable via Tailscale.

**Root cause:** `labels.go:238` deliberately skips parsing the `org.overthinkos.tunnel` OCI label — tunnel is deploy.yml-only. When `ov config setup` creates a new instance, it writes ports/env/security to deploy.yml but does not copy tunnel from the base entry.

**Workaround:** After `ov config setup -i <name>`, manually edit `~/.config/ov/deploy.yml` to add `tunnel: {provider: tailscale, private: all}` to the instance entry, then re-run `ov config setup -i <name>` to regenerate the quadlet.

### Resolution

`tunnel` inherits from defaults (image -> defaults -> nil). The shorthand `tunnel: tailscale` defaults to `private: all` (all ports on tailnet). The shorthand `tunnel: cloudflare` defaults to `public: all`.

Source: `ov/tunnel.go` (`schemeTarget`, `tailscaleFlag`, `isTCPFamily`, `validTailscaleSchemes`, `validCloudflareSchemes`), `ov/validate.go` (`validateTunnel`), `ov/quadlet.go` (systemd integration).

## deploy.yml — Source of Truth

`~/.config/ov/deploy.yml` is the **source of truth** for per-machine deployment configuration (not checked into git). All deployment commands read from image labels + deploy.yml — no `image.yml` needed.

### How it gets populated

1. **`ov config`** automatically persists: workspace, ports, env (CLI -e), env_file, network, security (auto-detected devices), volume backing (--bind/--encrypt)
2. **`ov deploy import`** merges pre-provisioned config (tunnel, volumes, DNS) from files
3. **`ov remove`** cleans the entry (use `--keep-deploy` to preserve for re-config)

### Structure

```yaml
images:
  my-app:
    workspace: /home/user/project     # saved by ov config
    tunnel:
      provider: cloudflare
      port: 2283
    fqdn: "app.example.com"
    volumes:
      - name: data
        type: bind
        host: "~/data/myapp"
      - name: secrets
        type: encrypted
    ports:
      - "2283:2283"
    env:
      - "LOG_LEVEL=debug"
    env_file: "/home/user/project/.env"
    security:
      devices:
        - /dev/dri/renderD128
    network: ov
    engine: podman
```

Allowed fields: `workspace`, `version`, `status`, `info`, `tunnel`, `fqdn`, `acme_email`, `volumes`, `ports`, `env`, `env_file`, `security`, `network`, `engine`, `secrets`, `target`, `add_layers`, `install_opts` (new fields described below).

### New fields from the `ov deploy add` surface

The `ov deploy add`/`del` refactor introduced three fields on every deploy.yml image entry. They're honored only when relevant to the deploy target.

**`target:`** — `""` or `"container"` (default, existing pipeline) or `"host"` (local filesystem apply). The deploy name `host` typically also sets this explicitly for clarity. When `target: host` is set on a non-`host` deploy name, `ov deploy add` errors cleanly — container deploy names can't secretly target the host.

**`add_layers:`** — list of extra layer refs applied on top of the image's base layers. Each entry accepts the same 4 ref forms as the command-line `--add-layer` flag (local name / local YAML path / remote `github.com/.../layers/<n>[@ref]`). See "add_layers: overlay mechanism" above for container vs host semantics.

**`install_opts:`** — host-target defaults that mirror the CLI flags on `ov deploy add`. CLI flags win on conflict; deploy.yml provides defaults so you don't have to repeat `--with-services --allow-repo-changes` on every invocation.

```yaml
images:
  host:
    image: fedora-coder
    target: host
    add_layers:
      - my-team-vimrc                                     # local layer
      - github.com/team-acme/configs/layers/sshkeys       # remote layer
      - ./private-overlay.yml                             # local file
    install_opts:
      with_services: true
      allow_repo_changes: true
      allow_root_tasks: false
      skip_incompatible: false
      verify: true
      builder_image: fedora-builder:2026.04
    env:
      OVERTHINK_DEV: "true"
```

Fields ignored on container deploys: `install_opts` (host-only). Fields ignored on host deploys: `volumes`, `ports`, `tunnel`, `sidecars`, `security`'s container-runtime bits.

### Resource Caps

Cgroup memory and CPU limits are stored in the `security:` block of deploy.yml and persist across `ov config` re-runs (a `--memory-max` flag applied once stays in effect until explicitly changed). The fields are:

```yaml
images:
  selkies-desktop:
    security:
      shm_size: "1g"
      memory_max: "6g"
      memory_high: "5g"
      memory_swap_max: "2g"
      cpus: "4.0"
```

**Merge semantics** (authoritative, from `ov/security.go`):

| Source | Merge rule |
|---|---|
| Layer → layer | Smallest value wins (tightest cap is the safer default) |
| Layers → image-level `security:` in image.yml | Image-level **replaces** the merged layer value |
| Image-level → deploy-level `security:` in deploy.yml | Deploy-level **replaces** the image-level value |
| CLI flag → deploy-level | CLI flag **writes** directly to deploy.yml (`--memory-max=...` on `ov config`) |

Quadlet emission (`[Service]` section of `.container` file):

- `memory_max` → `MemoryMax=6G` (lowercase `g` is auto-normalized to `G` because systemd parses lowercase as `infinity` — see `/ov-layers:chrome` gotcha)
- `memory_high` → `MemoryHigh=5G`
- `memory_swap_max` → `MemorySwapMax=2G`
- `cpus` → `CPUQuota=400%` (systemd percentage form: 1 core = 100%)

Direct-mode emission (podman run flags, for `engine.run=direct`): `--memory`, `--memory-reservation`, `--memory-swap`, `--cpus`. `SecurityArgs` in `ov/security.go` emits both forms from the same source of truth.

**Unset fields pass through** — setting `--memory-max=6g` alone will not wipe an existing `shm_size` from deploy.yml. Only the fields you pass on the CLI get overwritten; everything else is preserved from the current deploy.yml state.

**Canonical consumer:** the chrome layer. See `/ov-layers:chrome` (Resource Caps & Circuit Breaker) for the pattern that pairs these caps with supervisord's `chrome-crash-listener` event listener to trigger container rebuild on FATAL state — the only way to release orphan memfd shmem across a Chrome crash loop. See `/ov-layers:supervisord` (Event Listeners) for the event listener pattern in full and `/ov:layer` (Security Declaration) for the authoring side.

### Provides (Top-Level)

The `provides:` section holds all resolved env and MCP provides entries from deployed images. Managed automatically by `ov config` when images with `env_provides` or `mcp_provides` layers are deployed.

```yaml
provides:
  env:
    - name: OLLAMA_HOST
      value: http://ov-ollama:11434
      source: ollama
    - name: PGHOST
      value: ov-postgresql
      source: postgresql
  mcp:
    - name: jupyter
      url: http://ov-jupyter:8888/mcp
      transport: http
      source: jupyter
images:
  my-app: { ... }
```

- `provides.env:` — resolved env_provides entries with `{name, value, source}` (self-excluded per consumer)
- `provides.mcp:` — resolved mcp_provides entries with `{name, url, transport, source}` (pod-aware, no self-exclusion)
- `source` tracks which image injected each entry — used for cleanup on `ov remove`
- Priority for env (last wins): provides.env < per-image deploy env < deploy env_file < workspace .env < CLI --env-file < CLI -e
- `ov config remove` / `ov remove` automatically cleans up entries from the removed image
- Instance-aware cleanup: removing an instance (e.g., `ov remove selkies-desktop -i work`) only cleans provides entries sourced from that specific instance (`selkies-desktop/work`), not from other instances of the same base image. Base image removal requires no other instances to exist before cleaning provides

See `/ov:layer` for `env_provides`/`mcp_provides` field declarations and `/ov:config` for `--update-all` propagation.

### Secrets

Per-deployment secret source overrides. Secrets declared in image labels (from `layer.yml`) are provisioned as Podman secrets at `ov config` time. Deploy.yml can override where the value comes from:

```yaml
secrets:
  - name: api-key               # matches layer secret name
    source: keyring              # "keyring" (default), "env:VAR", "file:/path"
```

If no source is specified, the credential resolution chain is used: env var > keyring > config file.

### Workspace recall

Volume binding is configured at deploy time via `--bind` flags. The binding is persisted in deploy.yml:

```bash
ov config my-app --bind workspace=~/project    # Saves volume config to deploy.yml
ov remove my-app --keep-deploy                 # Quadlet removed, config preserved
ov config my-app                               # Picks up volumes from deploy.yml
```

### Deploy status audit

```bash
ov deploy status
# openclaw-ollama-sway-browser  deploy.yml: yes  quadlet: yes  (ok)
# old-service                   deploy.yml: yes  quadlet: no   (stale config)
# manual-service                deploy.yml: no   quadlet: yes  (no overrides)
```

### Labels-only architecture

Deployment commands (`ov config`, `start`, `status`, `logs`, `update`, `remove`, `seed`, `service`) resolve all configuration from **OCI image labels** + **deploy.yml** — no `image.yml` dependency. This means you can deploy on any machine with just `ov image pull` + `ov config`.

**Local-storage requirement.** Because deploy-mode commands read OCI labels directly from local container storage (via `ExtractMetadata` → `podman inspect`), the image must be pulled first. If it isn't, the command fails with `ErrImageNotLocal` and the CLI suggests `ov image pull`. See `/ov:pull` for the sentinel pattern and remote-ref (`@github.com/...`) handling.

**`MergeDeployOntoMetadata` ordering gotcha.** When extending deploy-mode code, remember that deploy-overlay fields like `meta.Tunnel` are nil until `MergeDeployOntoMetadata` runs. A `if meta.Tunnel != nil` check before the merge is unreachable code — this was the actual bug fixed in `start.go` by the refactor.

### Instance Support

Deploy multiple containers of the same image with `-i <instance>`:

```bash
ov config selkies-desktop -i work -e TS_HOSTNAME=work -p 3001:3000
ov config selkies-desktop -i personal -p 3002:3000
ov start selkies-desktop -i work
ov start selkies-desktop -i personal
```

**Deploy key convention:** Base images use `selkies-desktop` as the deploy.yml key. Instances use `selkies-desktop/work` (slash-separated). Functions: `deployKey()` constructs keys, `parseDeployKey()` splits them back. Source: `ov/deploy.go`.

**Container naming:** `ov-<image>-<instance>` (e.g., `ov-selkies-desktop-work`). Quadlet file: `ov-selkies-desktop-work.container`.

**Deploy.yml structure with instances:**

```yaml
images:
  selkies-desktop:
    ports: [3000:3000]
  selkies-desktop/work:
    ports: [3001:3000]
    env: [TS_HOSTNAME=work]
  selkies-desktop/personal:
    ports: [3002:3000]
```

**Instance lifecycle:** All commands accept `-i`: `ov start/stop/status/logs/remove <image> -i <instance>`, `ov deploy show/reset <image> -i <instance>`. Removing an instance only cleans its deploy.yml entry — the base and other instances are unaffected. Provides cleanup waits until the last entry for a base image is removed.

**Instance removal gotcha:** `ov config remove` disables the systemd service but does NOT remove the deploy.yml entry. You MUST also run `ov deploy reset <image> -i <instance>` and delete the quadlet file. If you run `ov config --update-all` before cleaning deploy.yml, stale quadlet files will be re-created. See `/ov:config` for the full 3-step cleanup workflow.

**MCP name disambiguation:** When an instance provides MCP servers, the server name gets `-<instance>` appended (e.g., `chrome-devtools-work`). See `/ov:config` for details.

## Volume Backing

Layers declare what persistent storage they need via `volumes:` in `layer.yml`. By default, all volumes are Docker/Podman named volumes. At `ov config` time, any volume's backing can be changed to a host bind mount or encrypted gocryptfs mount.

### Per-Volume Configuration via `ov config`

```bash
# Default: all volumes as named volumes (no flags needed)
ov config immich

# Configure specific volumes as bind mounts
ov config immich --bind import --bind external

# Bind mount with explicit host path
ov config immich --bind library=/mnt/nas/photos

# Configure volume as encrypted (gocryptfs)
ov config immich --encrypt library

# Canonical syntax: --volume name:type[:path]
ov config immich -v library:bind:/mnt/nas -v import:bind -v cache:encrypted

# Fully automated via env vars (no prompts)
OV_VOLUMES_IMMICH="library:bind:/mnt/nas,import:bind" ov config immich --password auto
```

### deploy.yml Volume Config

Volume backing choices are persisted in deploy.yml:

```yaml
volumes:
  - name: library
    type: bind
    host: "/mnt/nas/photos"     # explicit host path
  - name: import
    type: bind                   # no host → auto path: <volumes_path>/<image>/import
  - name: cache
    type: encrypted              # gocryptfs managed
```

**Fields:**
- `name`: matches a layer-declared volume name
- `type`: `volume` (default, named volume), `bind` (host directory), `encrypted` (gocryptfs)
- `host`: explicit host path — for `bind` type (optional, omit for auto path); for `encrypted` type, the direct volume directory containing `cipher/` and `plain/` (optional, omit to use global `encrypted_storage_path` with `ov-<image>-<name>` prefix)
- `path`: container path (only for deploy-only volumes not declared in any layer)
- `data_seeded`: `bool` — tracks whether data from image data layers was provisioned (set by `ov config`)
- `data_source`: `string` — image:tag that provided the data (updated by `ov config` and `ov update`)

**Auto path:** When `type: bind` and no `host` is specified, the host path is computed at runtime: `<volumes_path>/<image>/<name>`. Default volumes_path: `~/.local/share/ov/volumes/`. Configurable: `ov settings set volumes_path /mnt/nas/ov` (env: `OV_VOLUMES_PATH`).

**Unconfigured volumes** remain named volumes — no deploy.yml entry needed.

### Resolution Flow

`ResolveVolumeBacking()` in `ov/deploy.go` splits image volumes into named volumes and bind-backed mounts:

1. Load all volumes from image labels (`org.overthinkos.volumes`)
2. Load deploy.yml volume overrides for this image
3. For each declared volume:
   - If deploy.yml says `type=bind` → host bind mount (explicit path or auto path)
   - If deploy.yml says `type=encrypted` → gocryptfs FUSE mount
   - Otherwise → named volume (Docker/Podman-managed)
4. Deploy-only volumes (with `path:` set, not in any layer) are also supported

### Integration

- **Data provisioning**: `ov config` automatically provisions data from data layers into bind-backed volumes (via `--seed`, default true). `ov update` merges new data non-destructively. See `/ov:config` and `/ov:update`
- **`ov shell`/`ov start`**: resolves volume backing, verifies bind dirs exist and encrypted volumes are mounted, generates `-v` flags
- **`ov config` (quadlet)**: bind-backed volumes become `Volume=` lines with host paths. `--userns=keep-id` added when bind-backed volumes exist
- **`ov remove --purge`**: removes named volumes
- **`ov image inspect --format bind_mounts`**: outputs deploy-configured volume backing

Source: `ov/deploy.go` (`DeployVolumeConfig`, `ResolveVolumeBacking`), `ov/enc.go` (`ResolvedBindMount`).

## VNC Password for Deployments

For images with wayvnc (VNC on tcp:5900), set a VNC password after enabling:

```bash
ov config openclaw-sway-browser
ov test vnc passwd openclaw-sway-browser --generate   # auto-generates password, prints to stdout
```

Or pre-set via settings before deployment:

```bash
ov settings set vnc.password.openclaw-sway-browser mysecret
ov config openclaw-sway-browser
# After container starts, run passwd to configure server-side auth:
ov test vnc passwd openclaw-sway-browser    # uses stored password (no prompt)
```

See `/ov:vnc` for full VNC authentication documentation.

## Port Relay Pattern

Some services (OpenClaw) bind only to loopback for security. The `port_relay` field in `layer.yml` creates a socat relay from the container's network interface to loopback, making the service accessible externally without weakening its security model.

```yaml
# In layer.yml
ports:
  - 18789
port_relay:
  - 18789
```

Requires the `socat` layer as a dependency. The relay runs as a `relay-<port>` service in the configured init system. See `/ov-layers:openclaw` for an example.

**Chrome CDP exception:** Chrome DevTools no longer uses `port_relay`. Chrome 146+ rejects connections with non-localhost Host headers, so a simple socat relay is insufficient. Instead, Chrome uses a `cdp-proxy` Python supervisord service that listens on `0.0.0.0:9222`, forwards to Chrome on `127.0.0.1:9223` with Host header rewriting, and rewrites response URLs (e.g., `webSocketDebuggerUrl`) with Content-Length correction. See `/ov-layers:chrome` and `/ov:cdp` for details.

## Provides Configuration

Global environment and MCP server injection for all deployed images. Stored in deploy.yml under `provides:`.

### Structure

```yaml
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
```

- `provides.env:` — resolved env_provides entries with `{name, value, source}`
- `provides.mcp:` — resolved mcp_provides entries with `{name, url, transport, source}`
- `source` field tracks which image contributed each entry (used for cleanup on `ov remove`)
- Entries resolved at `ov config` time from layer `env_provides:` and `mcp_provides:` declarations
- `GlobalEnvForImage()` in `provides.go` resolves both env and MCP provides for each consumer image
- Env provides: self-excluded (prevents own env_provides from overriding service bind addresses)
- MCP provides: pod-aware (same-container entries resolve to `localhost`, no self-exclusion)
- Consumer containers receive `OV_MCP_SERVERS` JSON env var with resolved MCP server entries

See `/ov:config` for setup workflow and `/ov:layer` for declaration format.

## Sidecar Pod Deployment

When sidecars are attached via `ov config --sidecar <name>`, deployment generates a Podman **pod** instead of a standalone container. See `/ov:sidecar` for full sidecar documentation.

### Generated Files

| File | Content |
|------|---------|
| `ov-<image>.pod` | Pod: `Network=ov`, `PodmanArgs=-p` (ports), `--shm-size` |
| `ov-<image>-<sidecar>.container` | Sidecar: image, env, caps, devices, secrets |
| `ov-<image>.container` | App: `Pod=ov-<image>.pod`, no ports/network |

The pod owns the shared network namespace. Port mappings and ShmSize move from the container to the pod. The app container gets `Pod=` and loses `PublishPort=` and `Network=`.

### Dual Networking (Tailscale)

When a Tailscale sidecar is attached, the pod has dual networking:
- **"ov" bridge** — container-to-container connectivity, `env_provides` discovery
- **Tailscale tun** — exit node routing for outbound internet traffic
- `--exit-node-allow-lan-access` exempts bridge subnets from the tunnel

Host `tunnel: tailscale` (ExecStartPost=tailscale serve) and the sidecar are **independent**: the host tunnel serves ports on the host's tailnet, while the sidecar handles exit node routing on a potentially different tailnet.

### deploy.yml Sidecar Config

```yaml
images:
  selkies-desktop:
    sidecars:
      tailscale:
        env:
          TS_HOSTNAME: selkies-desktop
          TS_EXTRA_ARGS: "--exit-node=100.80.254.4 --exit-node-allow-lan-access"
```

## Cross-References

**Deploy surface (new):**
- `/ov:host-deploy` — Host-target execution model: HostDeployTarget, ledger, gates, 15 ReverseOp kinds, sudo batching
- `/ov-dev:install-plan` — The InstallPlan IR shared by `ov image build` (OCITarget), container deploys (ContainerDeployTarget), and host deploys (HostDeployTarget)
- `/ov-dev:host-infra` — Supporting Go files for host deploys: hostdistro, ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref

**Deploy-adjacent commands:**
- `/ov:pull` — Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path
- `/ov:sidecar` — Sidecar containers, pod networking, Tailscale exit nodes, Environment Contract (provides filtering)
- `/ov:service` — Service lifecycle (start/stop/update/remove)
- `/ov:start` — Ergonomic alias for `ov deploy add <image> <image>` (container target)
- `/ov:stop` — Ergonomic alias for `ov deploy del <name>`
- `/ov:update` — Per-instance update pattern; equivalent to `ov deploy add <name> --pull`
- `/ov:config` — Resource cap flags (`--memory-max/high/swap/cpus`), provides filtering, env_requires enforcement, NO_PROXY auto-enrichment, `--sidecar`, `-i` instance support, MCP name disambiguation
- `/ov:enc` — Encrypted storage commands (ov config mount/unmount)
- `/ov:vnc` — VNC password setup for desktop containers
- `/ov:vm` — Virtual machine deployment (ov vm)
- `/ov:build` — Building images before deployment (+ the `--no-cache` intermediate scratch-stage caveat)
- `/ov:mcp` — verify the MCP endpoints declared by `provides.mcp:` entries are actually reachable (`ov test mcp ping <image>`); note the **port-publishing gotcha** when a `ports:` override in deploy.yml predates a newly-added mcp-providing layer
- `/ov:image` — Image configuration, OCI label emission, `labels.go:238` tunnel read-skip
- `/ov:layer` — Unified `services:` schema (use_packaged + structured custom), `env_provides`/`env_requires`/`env_accepts` field declarations, security resource caps
- `/ov:test` — Local `tests:` in deploy.yml overlays image-baked deploy defaults: entries with matching `id:` replace, otherwise append. `id: X, skip: true` disables a baked check without a replacement.

**Canonical layer worked examples:**
- `/ov-layers:chrome` — Resource caps consumer + crash-loop circuit breaker
- `/ov-layers:supervisord` — Event listener pattern triggered by the caps; ServiceSchemaDef that renders `services:` entries to supervisord INI
- `/ov-layers:postgresql` — Canonical `use_packaged:` entry (packaged unit reuse)
- `/ov-layers:ollama`, `/ov-layers:hermes` — Custom `services:` entries
- `/ov-layers:selkies-desktop` — Multi-instance proxy deployment, tunnel inheritance workaround

## When to Use This Skill

**MUST be invoked** when the task involves quadlet generation, tunnels, bind mounts, or deploy overlays. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** After `/ov:build`, before `/ov:service`.
Previous step: `/ov:build` (build the image). Next step: `/ov:service` (start, status, logs).
