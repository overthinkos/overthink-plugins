---
name: deploy
description: |
  MUST be invoked before any work involving: `charly deploy add`/`charly deploy del` commands, quadlet generation, volume backing, tunnels (Tailscale/Cloudflare), `add_candy:` overlay, or per-machine deploy overlays.
---

# Deploy - Deployment Configuration

## Schema overview

- **Dispatch via explicit `target:`** — `local | vm | pod | k8s` (short, matches charly command verbs).
- **Cross-ref fields on `DeploymentNode`** — `vm: <entity>` for target: vm, `box: <name>` for target: pod, `k8s: <name>` for target: k8s, `inside: <deploy>` for nested local-deploy. Deploy nesting uses `nested:`.
- **Disposability is a deploy property** — `DeploymentNode.Disposable` is the sole source of truth.
- **Resource arbitration is a deploy property** — `preemptible:` (holder; occupies exclusive host-resource token(s), may be gracefully stopped + restored) and `requires_exclusive:` (claimant; needs sole use) on `DeploymentNode` drive the arbiter (`charly preempt`). A fourth axis ORTHOGONAL to disposable/ephemeral/lifecycle. See "Preemptible resource arbitration" below + `/charly-internals:disposable`.
- **Disposable R10 test beds are `kind: eval` entities** (run via `charly eval run <bed>`), NOT `deploy:` entries — ecosystem-wide. The main repo's beds (`eval-pod`, `eval-k3s-vm`, `eval-sway-browser-vnc-pod`, …) AND every `box/<distro>` submodule's beds (the arch / cachyos / debian / ubuntu / fedora bootstrap-VM + pacstrap/debootstrap beds, in each submodule's config — its `charly.yml` + per-kind sibling files) are `kind: eval`. Repos ship NO `kind: deploy` test beds; the one `kind: deploy` exception is the cachyos submodule's `charly-cachyos` operator workstation profile (a profile, not a test bed). Operator deployments otherwise live in the per-host `~/.config/charly/deploy.yml`. See `/charly-eval:eval` "kind: eval beds".

## Overview

`charly deploy` is the parent verb for applying and tearing down deployments, plus managing `deploy.yml` overrides. The command family has two distinct surfaces:

1. **Execution verbs** — `charly deploy add <name>` / `charly deploy del <name>`. Apply or reverse a deployment. Four targets are dispatched by the `target:` field:
   - `target: local` → `LocalDeployTarget` on the local filesystem (or, with `inside: <deploy>`, via NestedExecutor into the referenced deployment). See `/charly-local:local-deploy`.
   - `target: vm` (+ `vm: <entity>`) → `VmDeployTarget` inside a running VM via SSH. See "VM target" section below and `/charly-internals:vm-deploy-target`.
   - `target: pod` (+ `box: <image>`) → `PodDeployTarget`: overlay Containerfile + quadlet/podman.
   - `target: k8s` (+ `k8s: <name>`) → Kustomize base/overlays tree. See `/charly-kubernetes:kubernetes`.
2. **Config-file management** — `charly deploy show/export/import/reset/path/status`. Read and mutate `~/.config/charly/deploy.yml` itself.

## Targets, one schema

The same `DeploymentNode` shape feeds every target (`pod`, `local`, `vm`, `k8s`, `android`) — authors describe *what the workload needs* (ports, volumes, env, security, tests); the generator per target decides *how K8s / quadlet / local apply / VM over SSH / Android apk-install* realizes it.

**`target: android`** installs a deploy's `add_candy:` candies' `apk:` packages
onto a `kind: android` device (an in-pod emulator or a remote/physical adb
endpoint) via `AndroidDeployTarget` — the Android analogue of `target: k8s`
emitting workloads onto a cluster. The cross-ref is `android: <device>`; apps
ride in on `add_candy:` (no apk-list field). Nested `pod → android` (the device
on its emulator pod) mirrors `vm → k8s`; a pod's android children deploy AFTER
`charly start` (use `--node-only` on the pod's deploy-add, then dotted-path
`charly deploy add pod.device`). See `/charly-eval:android`. K8s-specific choices (storage class, ingress class, cert issuer, secret backend) live in a **cluster profile** file (`~/.config/charly/clusters/<name>.yaml` or in-repo `clusters/<name>.yaml`), *not* in the deployment. This means one deployment spec targets dev/staging/prod clusters with zero schema changes — only the cluster profile differs.

`charly start` / `charly stop` remain as ergonomic wrappers: `charly start <image>` is equivalent to `charly deploy add <image> <image>` with the container target; `charly stop <name>` is `charly deploy del <name>`. New scripts should prefer the explicit `charly deploy add`/`charly deploy del` forms, especially when using `--add-candy` overlays or the `host` target.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Apply container deploy | `charly deploy add <name> <ref>` | Compile candies + build overlay if `add_candy:` present + run via quadlet |
| Apply local deploy | `charly deploy add <name> <ref>` (entry `target: local`) | Apply layers directly to local filesystem (see `/charly-local:local-deploy`) |
| Tear down deploy | `charly deploy del <name>` | Stop container + reverse ReverseOps (host) + ledger cleanup |
| Dry-run | `charly deploy add <name> <ref> --dry-run [--format=json]` | Print the InstallPlan without executing |
| Layer overlay | `charly deploy add <name> <ref> --add-candy <ref>` | Extra layer(s) applied on top; repeatable |
| Configure deployment | `charly config <image>` | Generate .container file + save deploy.yml |
| Configure instance | `charly config <image> -i <instance>` | Generate instance-specific quadlet + deploy entry |
| Configure volume backing | `charly config <image> --bind name` | Set volume as host bind mount |
| Provision data | `charly config <image> --seed` | Auto-provision data candies into bind mounts (default) |
| Deploy status | `charly deploy status` | Audit deploy.yml vs quadlet sync |
| Show overrides | `charly deploy show [image]` | Display deploy.yml contents |
| Show instance overrides | `charly deploy show <image> -i <instance>` | Display instance-specific overrides |
| Import config | `charly deploy import <files>` | Merge files into deploy.yml |
| Reset config | `charly deploy reset [image]` | Remove deploy.yml overrides |
| Reset instance config | `charly deploy reset <image> -i <instance>` | Remove instance overrides |
| Push to registry | `charly box build --push` | Multi-platform push |

For service lifecycle commands (start/stop/status/logs/update/remove), see `/charly-core:service`. For VM lifecycle (build/create/start/stop/ssh), see `/charly-vm:vm`; for in-VM layer deploys via `charly deploy add vm:<name>`, see the "VM target" section below and `/charly-internals:vm-deploy-target`. For encrypted storage, see `/charly-automation:enc`. For host-target semantics, see `/charly-local:local-deploy`. For Kubernetes targets, see `/charly-kubernetes:kubernetes`. For the Go IR that drives all four targets, see `/charly-internals:install-plan`.

## Command Family: `add` / `del`

### `charly deploy add <name> [<ref>]`

Applies a deployment. The deploy entry's `target:` field selects the target:

- **`target: local`** — apply layers to the local filesystem via `LocalDeployTarget`. With `host: local` (default) the apply runs directly; with `host: <user@machine>` it runs over SSH. See `/charly-local:local-deploy`.
- **`target: vm`** (+ `vm: <entity>`) — apply layers inside a running `kind: vm` entity via SSH (`VmDeployTarget`). `<vm-name>` must match an entry in `vm.yml`; the VM must already be created (`charly vm create <vm-name>`). See "VM target" section below.
- **`target: k8s`** (+ `k8s: <name>`) — emit a Kustomize base/overlays tree. See `/charly-kubernetes:kubernetes`.
- **`target: pod`** (default, + `box: <image>`) — container deployment. Multiple pod deploys coexist (`my-dev`, `postgres-staging`, etc.); each gets its own quadlet, container name, and deploy.yml entry.

`<ref>` accepts four forms, auto-detected:

| Form | Example | Resolution |
|---|---|---|
| Local image name | `fedora-coder` | Looked up in current project's `charly.yml` |
| Local layer name | `pre-commit` | Looked up in current project's `candy/` directory |
| Local YAML path | `./custom.yml`, `/abs/path/charly.yml` | File's top-level keys classify image vs layer |
| Remote repo | `github.com/owner/repo[/box/<n>\|/candy/<n>][@ref]` | Fetched via existing `--repo` cache |

Disambiguation: a ref containing `/candy/` resolves to a candy; `/box/` to a box. For local names, `box/` is checked before `candy/`; same-named entries in both are a hard error. The legacy `@host/org/repo:version` form (used by `require:` and `candy:` in charly.yml) is also accepted.

When `<ref>` is omitted, the ref falls back to `deploy.yml['deploys'][<name>]['image']` (or the deploy key itself if no explicit image is declared).

### `charly deploy add` flags

**Universal:**
- `--tag <calver>` — override deploy.yml tag
- `--dry-run` — print the InstallPlan without executing
- `--format table|json` — with `--dry-run`
- `--pull` — force re-fetch of remote refs / image pull
- `--verify` — run layer `eval:` post-deploy
- `--add-candy <ref>` — repeatable; extra layer(s) applied on top (all 4 ref forms)

**Local-target-specific** (silently ignored on pod deploys):
- `--with-services` — opt-in for systemd unit installation (packaged-unit enable + drop-ins)
- `--allow-repo-changes` — opt-in for repo config mutations (rpmfusion, copr, external `.repo` files)
- `--allow-root-tasks` — opt-in for arbitrary `cmd: user: root` task bodies
- `--skip-incompatible` — skip layers lacking a host-matching format section instead of failing
- `--builder-image <ref>` — override the compile-builder image
- `--yes` / `-y` — all three gates plus skip sudo preflight

### `charly deploy del <name>`

Reverses a deployment. Gated host-side reversal respects `--keep-repo-changes` and `--keep-services`. Container teardown: `podman stop` + `rm` + overlay image removal (unless `--keep-image`) + ledger cleanup. VM teardown: SSH-executed ReverseOps in the guest, preserving the VM itself (use `charly vm destroy` separately). See `/charly-local:local-deploy` for the full 15-kind ReverseOp table.

## VM target: `charly deploy add vm:<vm-name> <ref>`

Applies layer recipes **inside a running VM** over SSH. Same `InstallPlan` IR as host and container targets; the difference is that bash bodies run via `ssh guest 'sudo bash -s'` through an `SSHExecutor` (see `/charly-internals:vm-deploy-target`).

**Prerequisite**: the VM must exist before `charly deploy add vm:...` runs. `charly deploy add vm:<name>` does NOT auto-provision; a missing VM produces a clean error pointing at `charly vm create`:

```bash
charly vm create arch                            # provision VM first
charly deploy add vm:arch ripgrep                # then apply layer in guest
charly deploy add vm:arch fedora-coder \
    --add-candy team-extras \
    --add-candy github.com/team/configs/candy/sshkeys
charly deploy del vm:arch                        # reverse all applied layers (VM stays up)
```

### `VmDeployState` schema in deploy.yml

When `charly deploy add vm:<name>` completes, a `vm_state` sub-object lands in the deploy.yml entry:

```yaml
deploy:
  vm:arch:
    target: vm
    vm_source:                             # persisted copy of VmSpec.Source for re-apply
      kind: cloud_image
      url: https://fastly.mirror.pkgbuild.com/...
      base_user: arch
    add_candy:
      - ripgrep
      - github.com/team/configs/candy/sshkeys
    install_opts:
      verify: true
    vm_state:
      instance_id: 7b3a8f42-...             # stable UUID across rebuilds
      nvram_path: ""                         # empty for firmware=bios
      last_build: 2026-04-22T10:14:27Z
      last_deploy: 2026-04-22T10:18:55Z
      applied_layers: [ripgrep, sshkeys]
      base_image_sha256: a8c9e0f1...         # cloud_image integrity trace
```

`vm_state` is persisted so re-applies pick up `instance_id` (cloud-init uses it as a stable identifier) and so `charly deploy del vm:<name>` knows which NVRAM to use. SSH access uses the managed `charly-<vmname>` ssh-config alias written by `charly vm create`.

### `target: vm` explicit declaration vs `vm:` prefix

Two ways to mark a deploy entry as VM-targeted:

1. **Name prefix (preferred)**: the deploy key is `vm:<vm-name>`. CLI dispatch reads the prefix and routes through `ResolveTarget` → `VmUnifiedTarget.Add`.
2. **Explicit field**: the deploy key is anything + `target: vm`. Useful when you want a non-prefixed deploy name (e.g., for readability in `deploy.yml`).

Using both is redundant but harmless. Using `target: vm` on a non-`vm:`-prefixed deploy whose underlying VM doesn't exist errors at `charly deploy add` time.

### `add_candy:` overlay semantics for VM targets

When `charly deploy add vm:<name> <ref>` runs with `--add-candy`, the extra layers are applied **inside the guest** alongside the primary ref. The compiler merges `<ref>` + `add_candy:` into a single topo-sorted `InstallPlan`; `VmDeployTarget.Emit` walks it over SSH. The guest-side ledger records both the base and overlay layers so `charly deploy del vm:<name>` reverses the full set.

This is the same merge semantics as `LocalDeployTarget` — just with SSH-wrapped execution. See `/charly-internals:install-plan` for the compiler and `/charly-internals:vm-deploy-target` for the execution model.

### VM-relevant `install_opts:` fields

`install_opts:` in the deploy.yml entry mirrors the CLI flags on `charly deploy add`. For VM targets, the relevant ones are:

| Field | Effect |
|---|---|
| `with_services` | Enable systemd units declared in layers' `service:` lists |
| `allow_repo_changes` | Permit repo-config mutations (rpmfusion, copr) in the guest |
| `allow_root_tasks` | Permit arbitrary `cmd: user: root` tasks in the guest |
| `verify` | Run layer `eval:` over SSH post-deploy |
| `skip_incompatible` | Skip layers lacking a guest-matching format section |

`builder_image` + `yes` also apply. Host-target-only gates that don't apply to VM target: none — the gate semantics are identical.

### Prerequisite chain

```
vm.yml declares kind:vm entity
    → charly vm build <name>         (cloud_image: fetch+resize+seed ISO; bootc: install to-disk)
    → charly vm create <name>        (libvirt domain + SMBIOS ssh key + passt portForward)
    → charly vm start <name>         (boot)
    → charly deploy add vm:<name> <ref>  (SSH → cloud-init wait → charly install → layer apply)
    → charly deploy del vm:<name>    (SSH → ReverseOps; VM stays up)
    → charly vm destroy <name>       (remove libvirt domain; --disk to also delete qcow2)
```

See `/charly-vm:vms-catalog` for vm.yml authoring, `/charly-vm:vm` for the lifecycle commands, `/charly-internals:vm-deploy-target` for the Emit flow.

### `add_candy:` overlay mechanism

Both container and host targets accept extra candies at deploy time via `--add-candy <ref>` (repeatable) or a `deploy.yml['deploys'][<name>]['add_layers']` list. Semantics:

- **Container target**: synthesizes an overlay Containerfile (`FROM <base-image>` + the extra candies' build steps) and builds a deterministic overlay image tagged `<deploy-name>-overlay:<short-hash>`. The deploy runs the overlay, not the base image. Re-running with different overlays rebuilds. `charly deploy del <name>` removes the overlay unless `--keep-image`.
- **Host target**: the compiler merges the box's candies with `add_candy:`, topo-sorts the union, and compiles one `InstallPlan` covering the combined set. The ledger records which candies (base + overlay) were applied so teardown reverses everything.

Ref forms for `--add-candy` are identical to the primary `<ref>` positional (local name / local path / remote / legacy `@` form).

## Two supported deploy patterns

Every `target: pod` deploy entry MUST declare its `box:` field
explicitly (hard load-time error if absent — see "Why `box:` is
required" below). With that contract locked, two distinct patterns
are supported:

### Pattern A — Multiple instances of the same image

Use the `<base>/<instance>` deploy-key form when you want N
container instances backed by the same image (one per tenant,
per workspace, per environment, …):

```yaml
deploy:
  versa:                       # base instance — deploy key == image name (just convention)
    box: versa
    target: pod

  versa/ecovoyage:             # `<base>/<instance>` deploy key
    box: versa               # SAME image; explicit (no inference)
    target: pod
    volume:
      - name: workspace
        type: bind
        host: /home/atrawog/Sync/Atrapub/ecovoyage
    port:
      - "32718:2718"           # explicit pinned ports for stable bookmarks
      - ...

  versa/another-tenant:        # second instance of versa
    box: versa
    target: pod
    volume:
      - name: workspace
        type: bind
        host: /srv/another-tenant/workspace
```

Container names: `charly-versa`, `charly-versa-ecovoyage`,
`charly-versa-another-tenant`. Equivalent CLI invocations:
`charly update versa -i ecovoyage` ↔ `charly update versa/ecovoyage` (the
`-i <inst>` flag and the `/` suffix are interchangeable for
addressing instance deploys).

### Pattern B — Arbitrary deploy name with image (and optional version pin)

The deploy key is purely a name; it does NOT have to match the
`box:` value. Use this for version pinning, canary deploys, or
when you want a more descriptive deploy name than the image name
itself:

```yaml
deploy:
  versa-prod:                                 # arbitrary deploy key
    box: versa                              # short name → resolves to current build
    target: pod

  versa-pinned-2026.131.2134:                 # pinned-version deploy
    box: ghcr.io/overthinkos/versa:2026.131.2134  # explicit ref, never re-resolved
    target: pod

  versa-canary:                               # canary deploy on a rolling tag
    box: ghcr.io/overthinkos/versa:next
    target: pod
```

Container names: `charly-versa-prod`, `charly-versa-pinned-2026.131.2134`,
`charly-versa-canary`. Only the single `<key>` form addresses these
(`charly update versa-prod`, NOT `charly update versa -i prod` — that would
target an instance of `versa`, which is a different deploy).

### Schema rules locked down by these patterns

- **`box:` is REQUIRED on every `target: pod` deploy entry.** Hard
  load-time error if absent, with a remediation hint pointing at
  `charly migrate` (the one-shot migration that injects
  the field into legacy entries).
- **The `box:` value is either**:
  - a **short name** (e.g. `versa`) — resolved against `box:`
    entries in `charly.yml` to the currently-built tag, OR
  - a **fully-qualified registry ref** (e.g.
    `ghcr.io/overthinkos/versa:2026.131.2134` or
    `…@sha256:…`) — pinned to that exact image, never re-resolved.
    Use this for version pinning and canary deploys.
- **The deploy key is purely a name.** It may contain `/` to express
  the multi-instance pattern (`<base>/<instance>`) but does NOT have
  to match `box:`. `versa-old`, `versa-canary`, and
  `my-tenant/staging` are all valid deploy keys.
- **Container name rule**: `charly-<key-with-slash-replaced-by-dash>`
  (e.g. `charly-versa`, `charly-versa-ecovoyage`, `charly-versa-canary`).
- **`charly update <key>` and `charly update <base> -i <inst>`** are
  equivalent ways to address a `<base>/<inst>` deploy (Pattern A).
  For Pattern B (arbitrary name), only the single `<key>` form
  works.

### Port resolution is keyed by deploy-key, not image short-name

Each deploy entry's quadlet `PublishPort=` is sourced from THAT entry's own
`port:`/`resolved_port:`, looked up by its deploy key. Multiple deploys backed
by the same image short-name on one host (Pattern A instances, Pattern B
pinned/canary deploys, and `kind: eval` beds whose `box:` matches a running
production deploy) each get their own independent ports — a bed remapping
`45434:11434` keeps that mapping even while a sibling `ollama` deploy publishes
`11434`. `MergeDeployOntoMetadata` and `dc.Lookup` both take the deploy key
(typically `c.Image`), never a value derived from the baked
`ai.opencharly.box` label, so an entry's explicit `port:` is never clobbered
by a sibling that merely shares the image.

### Why `box:` is required (R10 implication)

The eval runner inspects exactly the image the operator declared in
`box:`, never what the container happens to be. Without an explicit
`box:` field the runner would resolve the running container's image
ref via `containerImageRef()`; for volume-pinned deploys (where
`data_source:` seeds workspace from a specific image tag) the running
container sits at the seed-version, not the current image, so the
runner would read the older OCI label and silently drop any probes
added since. Requiring `box:` makes the inspected image deterministic.

## Examples

**Deploy a local image as a container:**
```bash
charly deploy add my-dev fedora-coder
# Uses deploy.yml['deploys']['my-dev'] for volumes/ports/env/tunnel.
charly deploy del my-dev
```

**Deploy directly to the host:**
```bash
charly deploy add host fedora-coder --with-services --assume-yes
charly deploy del host --assume-yes
```

**Deploy from a remote repo:**
```bash
charly deploy add my-coder github.com/overthinkos/overthink/box/fedora-coder@main
charly deploy add host github.com/team-acme/private-configs/candy/my-team-tools
```

**Add overlay layers:**
```bash
charly deploy add host fedora-coder \
    --add-candy team-extras \
    --add-candy github.com/team/configs/candy/sshkeys \
    --add-candy ./private.yml \
    --with-services
```

**Dry-run to preview the plan:**
```bash
charly deploy add host fedora-coder --dry-run --format=json
```

**`charly start`/`charly stop` equivalence:**
```bash
charly start fedora-coder            # == charly deploy add fedora-coder fedora-coder (container target)
charly stop fedora-coder             # == charly deploy del fedora-coder
```

## Quadlet Generation

User-level systemd services via podman quadlet. Generated by `charly config`.

### Generated File

Path: `~/.config/containers/systemd/charly-<image>.container` (or `charly-<image>-<instance>.container` with `-i`).

Contents include:
- `[Container]` section: image reference, container name, port mappings, volumes, environment
- `[Service]` section: restart policy, lifecycle hooks
- `[Install]` section: `WantedBy=default.target` (omitted for encrypted services without keyring backend)
- `PodmanArgs=` for security settings (privileged, capabilities, devices)
- `Volume=` for named volumes and plain bind mounts
- `Environment=` / `EnvironmentFile=` for env vars
- `ExecStartPost=` / `ExecStopPost=` for tunnel commands

Service name: `charly-<image>.service`. Container name: `charly-<image>`. Entrypoint: determined by build.yml `init:` section for the configured init system. Encrypted volumes are mounted via `ExecStartPre=charly config mount` in the quadlet, which creates transient `charly-enc-<image>-<volume>.scope` units for each encrypted volume. These scope units are independent of the container service — they survive stop/restart (see `/charly-automation:enc`). With Secret Service backend: auto-starts after login (ExecStartPre waits for keyring unlock, `TimeoutStartSec=0`). With KeePass or no backend: requires `charly start` (no `WantedBy=default.target`).

### Security in Quadlet

Layer and image-level security settings become `PodmanArgs=` in the quadlet file:

- `privileged: true` -> `PodmanArgs=--privileged`
- `cap_add` -> `PodmanArgs=--cap-add=<CAP>`
- `devices` -> `PodmanArgs=--device=<DEV>`
- `security_opt` -> `PodmanArgs=--security-opt=<OPT>`

Source: `charly/security.go`, `charly/quadlet.go`.

### Image Transfer

When `engine.build=docker`, `charly config` auto-detects if the image is missing from podman and transfers via `docker save | podman load`. `charly update` re-transfers if needed.

Source: `charly/quadlet.go` (generation), `charly/commands.go` (command structs).

## Tunnel Configuration

Expose services outside the container host via tunnels. Tunnel config lives exclusively in `deploy.yml` — it is NOT in `charly.yml` or OCI image labels. `charly config setup` persists tunnel config automatically via `saveDeployState`.

### Tailscale Serve (tailnet-private, default)

Exposes a port to your Tailscale network only. No FQDN needed -- Tailscale handles TLS automatically. Any port works for tailnet-only serve.

```yaml
tunnel: tailscale
# or expanded:
tunnel:
  provider: tailscale
  private: all       # all image ports on tailnet
```

**`bind_address` must be `127.0.0.1` (the default).** Setting `0.0.0.0` causes the container to bind on the Tailscale interface, preventing Tailscale from intercepting TLS. Result: HTTPS fails with `wrong version number`.

**Port form in `deploy.yml`.** The canonical form is bare `H:C` (e.g. `8888:8888`); `charly config` prepends `127.0.0.1:` automatically when a tunnel is set. The IP-prefixed form `127.0.0.1:8888:8888` (and IPv6 `[::1]:8888:8888`) is also accepted — the canonical `ParsePortMapping` helper in `charly/ports.go` normalizes both shapes to a single-prefixed `PublishPort=` line, so neither form produces a doubled `127.0.0.1:127.0.0.1:8888:8888` quadlet. Unparseable port strings are logged loudly to stderr rather than silently dropped (a silent skip would otherwise suppress the entire `ExecStartPost=tailscale serve` block when even one port couldn't be parsed).

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
  tunnel: my-tunnel    # optional, defaults to charly-<image>
fqdn: "app.example.com"
```

`charly config` handles the full tunnel lifecycle automatically:
1. Creates the Cloudflare tunnel (`cloudflared tunnel create`) if it doesn't exist
2. Writes the tunnel config YAML (`~/.config/charly/tunnels/<name>.yml`) with ingress rules
3. Routes DNS with `--overwrite-dns` (creates or updates CNAME to tunnel)
4. Generates a companion systemd service (`charly-<image>-tunnel.service`)
5. Enables the tunnel service and adds `Wants=` to the container quadlet

`charly start` then starts both the container and the tunnel service together.

### Backend Schemes

Port protocols declared in `charly.yml` control the backend URL scheme used by tunnel commands. The protocol flows from layer → OCI label (`ai.opencharly.port_proto`) → tunnel command.

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

`charly box validate` checks port schemes against provider capabilities. For example, `ssh` is valid for Cloudflare but not Tailscale; `tls-terminated-tcp` is valid for Tailscale but not Cloudflare.

See `/charly-image:layer` for port protocol syntax in `charly.yml`.

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

Port protocols are stored in the `ai.opencharly.port_proto` image label so deploy-mode commands work without access to the original layer definitions. Remote refs require `charly box pull` first — see `/charly-build:pull`.

### Instance Tunnel Inheritance

**Critical:** When deploying instances with `charly config setup -i <name>`, tunnel config is NOT auto-inherited from the base image's deploy.yml entry. Each instance must have its own `tunnel:` section in deploy.yml. Without it, the generated quadlet will have no `ExecStartPost=tailscale serve` commands and the instance will be unreachable via Tailscale.

**Root cause:** `labels.go:238` deliberately skips parsing the `ai.opencharly.tunnel` OCI label — tunnel is deploy.yml-only. When `charly config setup` creates a new instance, it writes ports/env/security to deploy.yml but does not copy tunnel from the base entry.

**Workaround:** After `charly config setup -i <name>`, manually edit `~/.config/charly/deploy.yml` to add `tunnel: {provider: tailscale, private: all}` to the instance entry, then re-run `charly config setup -i <name>` to regenerate the quadlet.

### Resolution

`tunnel` inherits from defaults (image -> defaults -> nil). The shorthand `tunnel: tailscale` defaults to `private: all` (all ports on tailnet). The shorthand `tunnel: cloudflare` defaults to `public: all`.

Source: `charly/tunnel.go` (`schemeTarget`, `tailscaleFlag`, `isTCPFamily`, `validTailscaleSchemes`, `validCloudflareSchemes`), `charly/validate.go` (`validateTunnel`), `charly/quadlet.go` (systemd integration).

## deploy.yml — Source of Truth

`~/.config/charly/deploy.yml` is the **source of truth** for per-machine deployment configuration (not checked into git). All deployment commands read from image labels + deploy.yml — no `charly.yml` needed.

### How it gets populated

1. **`charly config`** automatically persists: workspace, ports, env (CLI -e), env_file, network, security (auto-detected devices), volume backing (--bind/--encrypt)
2. **`charly deploy import`** merges pre-provisioned config (tunnel, volumes, DNS) from files
3. **`charly remove`** cleans the entry (use `--keep-deploy` to preserve for re-config)

### Legacy-schema rejection

The top-level deploy map is `deploy:`, and per-entry storage uses a structured `volume:` list. **`yaml.Unmarshal` silently drops unknown root keys**, so a file with the obsolete `image:` root key would parse to an empty `DeployConfig.Deploy` map and downstream commands would behave as if nothing was deployed — including the dangerous case where `bind_mounts: [{encrypted: true}]` entries become invisible to `loadEncryptedVolumes` and the encryption guarantee silently disappears. `LoadDeployConfig` (`charly/deploy.go:hasLegacyImagesKey`) detects the legacy root shape and fails loud, pointing at `charly migrate`.

`charly status` surfaces this as a non-fatal warning (graceful degradation falls back to image-label-driven display); the strictly-deploy.yml-driven verbs (`charly deploy show`, `charly config status`, `charly start`) hard-fail. Run `charly migrate` to convert in place — it backs the original up to `<file>.bak.<unix-ts>` and rewrites to the latest schema. See `/charly-build:migrate` "charly migrate".

### Structure

```yaml
version: 2026.144.1443
deploy:
  my-app:
    target: pod
    tunnel:
      provider: cloudflare
      port: 2283
    dns: "app.example.com"
    volumes:
      - name: data
        type: bind
        host: "~/data/myapp"
      - name: workspace
        type: bind
        host: /home/user/project
        path: /workspace
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
    network: charly
    engine: podman
```

Allowed fields: `workspace`, `version`, `tunnel`, `fqdn`, `acme_email`, `volumes`, `ports`, `env`, `env_file`, `security`, `network`, `engine`, `secrets`, `target`, `add_layers`, `install_opts`.

### `target` / `add_layer` / `install_opts` fields

The `charly deploy add`/`del` surface carries three fields on every deploy.yml entry. They're honored only when relevant to the deploy target.

**`target:`** — `pod` (default, container pipeline) or `local` (local filesystem apply). When `target: local` is set, `charly deploy add` routes to the local executor; `host: local` (default) runs directly, `host: <user@machine>` runs over SSH.

**`add_candy:`** — list of extra layer refs applied on top of the image's base layers. Each entry accepts the same 4 ref forms as the command-line `--add-candy` flag (local name / local YAML path / remote `github.com/.../candy/<n>[@ref]`). See "add_candy: overlay mechanism" above for pod vs local semantics.

**`install_opts:`** — local-target defaults that mirror the CLI flags on `charly deploy add`. CLI flags win on conflict; deploy.yml provides defaults so you don't have to repeat `--with-services --allow-repo-changes` on every invocation.

```yaml
deploy:
  my-host:
    box: fedora-coder
    target: local
    add_candy:
      - my-team-vimrc                                     # local layer
      - github.com/team-acme/configs/candy/sshkeys       # remote layer
      - ./private-overlay.yml                             # local file
    install_opts:
      with_services: true
      allow_repo_changes: true
      allow_root_tasks: false
      skip_incompatible: false
      verify: true
      builder_image: fedora-builder:2026.04
    env:
      OPENCHARLY_DEV: "true"
```

Fields ignored on pod deploys: `install_opts` (local-only). Fields ignored on local deploys: `volumes`, `ports`, `tunnel`, `sidecars`, `security`'s container-runtime bits.

### Resource Caps

Cgroup memory and CPU limits are stored in the `security:` block of deploy.yml and persist across `charly config` re-runs (a `--memory-max` flag applied once stays in effect until explicitly changed). The fields are:

```yaml
deploy:
  selkies-desktop:
    security:
      shm_size: "1g"
      memory_max: "6g"
      memory_high: "5g"
      memory_swap_max: "2g"
      cpus: "4.0"
```

**Merge semantics** (authoritative, from `charly/security.go`):

| Source | Merge rule |
|---|---|
| Layer → layer | Smallest value wins (tightest cap is the safer default) |
| Layers → image-level `security:` in charly.yml | Box-level **replaces** the merged layer value |
| Box-level → deploy-level `security:` in deploy.yml | Deploy-level **replaces** the image-level value |
| CLI flag → deploy-level | CLI flag **writes** directly to deploy.yml (`--memory-max=...` on `charly config`) |

Quadlet emission (`[Service]` section of `.container` file):

- `memory_max` → `MemoryMax=6G` (lowercase `g` is auto-normalized to `G` because systemd parses lowercase as `infinity` — see `/charly-selkies:chrome` gotcha)
- `memory_high` → `MemoryHigh=5G`
- `memory_swap_max` → `MemorySwapMax=2G`
- `cpus` → `CPUQuota=400%` (systemd percentage form: 1 core = 100%)

Direct-mode emission (podman run flags, for `engine.run=direct`): `--memory`, `--memory-reservation`, `--memory-swap`, `--cpus`. `SecurityArgs` in `charly/security.go` emits both forms from the same source of truth.

**Unset fields pass through** — setting `--memory-max=6g` alone will not wipe an existing `shm_size` from deploy.yml. Only the fields you pass on the CLI get overwritten; everything else is preserved from the current deploy.yml state.

**Canonical consumer:** the chrome candy's cgroup caps. See `/charly-selkies:chrome` (Resource Caps) — the caps bound a Chrome crash loop's blast radius; a wedged loop (orphan memfd shmem) is cleared by restarting the container, which tears down the cgroup. See `/charly-infrastructure:supervisord` (Event Listeners) for the eventlistener pattern in general and `/charly-image:layer` (Security Declaration) for the authoring side.

### Provides (Top-Level)

The `provides:` section holds all resolved env and MCP provides entries from deployed images. Managed automatically by `charly config` when images with `env_provide` or `mcp_provide` layers are deployed.

```yaml
provides:
  env:
    - name: OLLAMA_HOST
      value: http://charly-ollama:11434
      source: ollama
    - name: PGHOST
      value: charly-postgresql
      source: postgresql
  mcp:
    - name: jupyter
      url: http://charly-jupyter:8888/mcp
      transport: http
      source: jupyter
deploy:
  my-app: { ... }
```

- `provides.env:` — resolved env_provide entries with `{name, value, source}` (self-excluded per consumer)
- `provides.mcp:` — resolved mcp_provide entries with `{name, url, transport, source}` (pod-aware, no self-exclusion)
- `source` tracks which image injected each entry — used for cleanup on `charly remove`
- Priority for env (last wins): provides.env < per-image deploy env < deploy env_file < workspace .env < CLI --env-file < CLI -e
- `charly config remove` / `charly remove` automatically cleans up entries from the removed image
- Instance-aware cleanup: removing an instance (e.g., `charly remove selkies-desktop -i work`) only cleans provides entries sourced from that specific instance (`selkies-desktop/work`), not from other instances of the same base image. Base image removal requires no other instances to exist before cleaning provides

See `/charly-image:layer` for `env_provide`/`mcp_provide` field declarations and `/charly-core:charly-config` for `--update-all` propagation.

### Secrets

Per-deployment secret source overrides. Secrets declared in image labels (from `charly.yml`) are provisioned as Podman secrets at `charly config` time. Deploy.yml can override where the value comes from:

```yaml
secrets:
  - name: api-key               # matches layer secret name
    source: keyring              # "keyring" (default), "env:VAR", "file:/path"
```

If no source is specified, the credential resolution chain is used: env var > keyring > config file.

### Workspace recall

Volume binding is configured at deploy time via `--bind` flags. The binding is persisted in deploy.yml:

```bash
charly config my-app --bind workspace=~/project    # Saves volume config to deploy.yml
charly remove my-app --keep-deploy                 # Quadlet removed, config preserved
charly config my-app                               # Picks up volumes from deploy.yml
```

### Deploy status audit

```bash
charly deploy status
# sway-browser-vnc              deploy.yml: yes  quadlet: yes  (ok)
# old-service                   deploy.yml: yes  quadlet: no   (stale config)
# manual-service                deploy.yml: no   quadlet: yes  (no overrides)
```

### Labels-only architecture

Deployment commands (`charly config`, `start`, `status`, `logs`, `update`, `remove`, `seed`, `service`) resolve all configuration from **OCI image labels** + **deploy.yml** — no `charly.yml` dependency. This means you can deploy on any machine with just `charly box pull` + `charly config`.

**Local-storage requirement.** Because deploy-mode commands read OCI labels directly from local container storage (via `ExtractMetadata` → `podman inspect`), the image must be pulled first. If it isn't, the command fails with `ErrImageNotLocal` and the CLI suggests `charly box pull`. See `/charly-build:pull` for the sentinel pattern and remote-ref (`@github.com/...`) handling.

**`MergeDeployOntoMetadata` ordering gotcha.** When extending deploy-mode code, remember that deploy-overlay fields like `meta.Tunnel` are nil until `MergeDeployOntoMetadata` runs. A `if meta.Tunnel != nil` check before the merge is unreachable code — this was the actual bug fixed in `start.go` by the refactor.

### Instance Support

Deploy multiple containers of the same image with `-i <instance>`:

```bash
charly config selkies-desktop -i work -e TS_HOSTNAME=work -p 3001:3000
charly config selkies-desktop -i personal -p 3002:3000
charly start selkies-desktop -i work
charly start selkies-desktop -i personal
```

**Deploy key convention:** Base images use `selkies-desktop` as the deploy.yml key. Instances use `selkies-desktop/work` (slash-separated). Functions: `deployKey()` constructs keys, `parseDeployKey()` splits them back. Source: `charly/deploy.go`.

**Container naming:** `charly-<image>-<instance>` (e.g., `charly-selkies-desktop-work`). Quadlet file: `charly-selkies-desktop-work.container`.

**Deploy.yml structure with instances:**

```yaml
deploy:
  selkies-desktop:
    ports: [3000:3000]
  selkies-desktop/work:
    ports: [3001:3000]
    env: [TS_HOSTNAME=work]
  selkies-desktop/personal:
    ports: [3002:3000]
```

**Instance lifecycle:** All commands accept `-i`: `charly start/stop/status/logs/remove <image> -i <instance>`, `charly deploy show/reset <image> -i <instance>`. Removing an instance only cleans its deploy.yml entry — the base and other instances are unaffected. Provides cleanup waits until the last entry for a base image is removed.

**Instance removal gotcha:** `charly config remove` disables the systemd service but does NOT remove the deploy.yml entry. You MUST also run `charly deploy reset <image> -i <instance>` and delete the quadlet file. If you run `charly config --update-all` before cleaning deploy.yml, stale quadlet files will be re-created. See `/charly-core:charly-config` for the full 3-step cleanup workflow.

**MCP name disambiguation:** When an instance provides MCP servers, the server name gets `-<instance>` appended (e.g., `chrome-devtools-work`). See `/charly-core:charly-config` for details.

## Volume Backing

Layers declare what persistent storage they need via `volume:` in `charly.yml`. By default, all volumes are Docker/Podman named volumes. At `charly config` time, any volume's backing can be changed to a host bind mount or encrypted gocryptfs mount.

### Per-Volume Configuration via `charly config`

```bash
# Default: all volumes as named volumes (no flags needed)
charly config immich

# Configure specific volumes as bind mounts
charly config immich --bind import --bind external

# Bind mount with explicit host path
charly config immich --bind library=/mnt/nas/photos

# Configure volume as encrypted (gocryptfs)
charly config immich --encrypt library

# Canonical syntax: --volume name:type[:path]
charly config immich -v library:bind:/mnt/nas -v import:bind -v cache:encrypted

# Fully automated via env vars (no prompts)
CHARLY_VOLUMES_IMMICH="library:bind:/mnt/nas,import:bind" charly config immich --password auto
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
- `host`: explicit host path — for `bind` type (optional, omit for auto path); for `encrypted` type, the direct volume directory containing `cipher/` and `plain/` (optional, omit to use global `encrypted_storage_path` with `charly-<image>-<name>` prefix)
- `path`: container path (only for deploy-only volumes not declared in any layer)
- `data_seeded`: `bool` — tracks whether data from image data candies was provisioned (set by `charly config`)
- `data_source`: `string` — image:tag that provided the data (updated by `charly config` and `charly update`)

**Auto path:** When `type: bind` and no `host` is specified, the host path is computed at runtime: `<volumes_path>/<image>/<name>`. Default volumes_path: `~/.local/share/charly/volumes/`. Configurable: `charly settings set volumes_path /mnt/nas/charly` (env: `CHARLY_VOLUMES_PATH`).

**Unconfigured volumes** remain named volumes — no deploy.yml entry needed.

### Resolution Flow

`ResolveVolumeBacking()` in `charly/deploy.go` splits image volumes into named volumes and bind-backed mounts:

1. Load all volumes from image labels (`ai.opencharly.volume`)
2. Load deploy.yml volume overrides for this image
3. For each declared volume:
   - If deploy.yml says `type=bind` → host bind mount (explicit path or auto path)
   - If deploy.yml says `type=encrypted` → gocryptfs FUSE mount
   - Otherwise → named volume (Docker/Podman-managed)
4. Deploy-only volumes (with `path:` set, not in any layer) are also supported

### Integration

- **Data provisioning**: `charly config` automatically provisions data from data candies into bind-backed volumes (via `--seed`, default true). `charly update` merges new data non-destructively. See `/charly-core:charly-config` and `/charly-core:charly-update`
- **`charly shell`/`charly start`**: resolves volume backing, verifies bind dirs exist and encrypted volumes are mounted, generates `-v` flags
- **`charly config` (quadlet)**: bind-backed volumes become `Volume=` lines with host paths. `--userns=keep-id` added when bind-backed volumes exist
- **`charly remove --purge`**: removes named volumes
- **`charly box inspect --format bind_mounts`**: outputs deploy-configured volume backing

Source: `charly/deploy.go` (`DeployVolumeConfig`, `ResolveVolumeBacking`), `charly/enc.go` (`ResolvedBindMount`).

## VNC Password for Deployments

For images with wayvnc (VNC on tcp:5900), set a VNC password after enabling:

```bash
charly config sway-browser-vnc
charly eval vnc passwd sway-browser-vnc --generate   # auto-generates password, prints to stdout
```

Or pre-set via settings before deployment:

```bash
charly settings set vnc.password.sway-browser-vnc mysecret
charly config sway-browser-vnc
# After container starts, run passwd to configure server-side auth:
charly eval vnc passwd sway-browser-vnc    # uses stored password (no prompt)
```

See `/charly-eval:vnc` for full VNC authentication documentation.

## Port Relay Pattern

Some services (OpenClaw) bind only to loopback for security. The `port_relay` field in `charly.yml` creates a socat relay from the container's network interface to loopback, making the service accessible externally without weakening its security model.

```yaml
# In charly.yml
ports:
  - 18789
port_relay:
  - 18789
```

Requires the `socat` layer as a dependency. The relay runs as a `relay-<port>` service in the configured init system. See `/charly-openclaw:openclaw` for an example.

**Chrome CDP exception:** Chrome DevTools no longer uses `port_relay`. Chrome 146+ rejects connections with non-localhost Host headers, so a simple socat relay is insufficient. Instead, Chrome uses a `cdp-proxy` Python supervisord service that listens on `0.0.0.0:9222`, forwards to Chrome on `127.0.0.1:9223` with Host header rewriting, and rewrites response URLs (e.g., `webSocketDebuggerUrl`) with Content-Length correction. See `/charly-selkies:chrome` and `/charly-eval:cdp` for details.

## Provides Configuration

Global environment and MCP server injection for all deployed images. Stored in deploy.yml under `provides:`.

### Structure

```yaml
provides:
  env:
    - name: OLLAMA_HOST
      value: http://charly-ollama:11434
      source: ollama
  mcp:
    - name: jupyter
      url: http://charly-jupyter:8888/mcp
      transport: http
      source: jupyter
```

- `provides.env:` — resolved env_provide entries with `{name, value, source}`
- `provides.mcp:` — resolved mcp_provide entries with `{name, url, transport, source}`
- `source` field tracks which image contributed each entry (used for cleanup on `charly remove`)
- Entries resolved at `charly config` time from layer `env_provide:` and `mcp_provide:` declarations
- `GlobalEnvForImage()` in `provides.go` resolves both env and MCP provides for each consumer image
- Env provides: self-excluded (prevents own env_provide from overriding service bind addresses)
- MCP provides: pod-aware (same-container entries resolve to `localhost`, no self-exclusion)
- Consumer containers receive `CHARLY_MCP_SERVERS` JSON env var with resolved MCP server entries

See `/charly-core:charly-config` for setup workflow and `/charly-image:layer` for declaration format.

## Sidecar Pod Deployment

When sidecars are attached via `charly config --sidecar <name>`, deployment generates a Podman **pod** instead of a standalone container. See `/charly-automation:sidecar` for full sidecar documentation.

### Generated Files

| File | Content |
|------|---------|
| `charly-<image>.pod` | Pod: `Network=charly`, `PodmanArgs=-p` (ports), `--shm-size` |
| `charly-<image>-<sidecar>.container` | Sidecar: image, env, caps, devices, secrets |
| `charly-<image>.container` | App: `Pod=charly-<image>.pod`, no ports/network |

The pod owns the shared network namespace. Port mappings and ShmSize move from the container to the pod. The app container gets `Pod=` and loses `PublishPort=` and `Network=`.

### Dual Networking (Tailscale)

When a Tailscale sidecar is attached, the pod has dual networking:
- **"charly" bridge** — container-to-container connectivity, `env_provide` discovery
- **Tailscale tun** — exit node routing for outbound internet traffic
- `--exit-node-allow-lan-access` exempts bridge subnets from the tunnel

Host `tunnel: tailscale` (ExecStartPost=tailscale serve) and the sidecar are **independent**: the host tunnel serves ports on the host's tailnet, while the sidecar handles exit node routing on a potentially different tailnet.

### deploy.yml Sidecar Config

```yaml
deploy:
  selkies-desktop:
    sidecars:
      tailscale:
        env:
          TS_HOSTNAME: selkies-desktop
          TS_EXTRA_ARGS: "--exit-node=100.80.254.4 --exit-node-allow-lan-access"
```

## Preemptible resource arbitration (`preemptible` / `requires_exclusive` / `charly preempt`)

A physical host resource can be held by only ONE deployment at a time — the
canonical case is a GPU passed through to a VM via VFIO. The resource arbiter
(`charly/preempt.go`) frees such a resource on demand and gives it back.

```yaml
deploy:
  gpu-workstation:                 # HOLDER — a long-running operator VM
    target: vm
    vm: gpu-workstation-vm
    preemptible:
      holds: [nvidia-gpu]          # exclusive-resource token(s) it occupies (shorthand: preemptible: [nvidia-gpu])
      stop: shutdown               # graceful shutdown (default & only) — frees a VFIO device
      restore: always              # always (default) | on-success

# CLAIMANT (a deploy or a kind:eval bed) that needs sole use while it runs:
eval:
  eval-gpu-bed:
    target: vm
    vm: gpu-eval-vm
    disposable: true
    requires_exclusive: [nvidia-gpu]
```

- **When the arbiter acts.** Before a claimant is brought up — `charly eval run <bed>`
  (transient claim, auto-released at teardown), or a standalone `charly vm create` /
  `charly start` (persistent claim, released on `charly vm stop`/`vm destroy`/`charly stop`/
  `charly remove`) — it gracefully stops every running preemptible holder whose
  `holds:` intersects the claimant's `requires_exclusive:`, waits for it to
  actually power off (so the resource is truly released), records a crash-safe
  lease, then proceeds. Nested `charly` subprocesses inherit the lease
  (`CHARLY_PREEMPT_LEASE`) and never re-acquire.
- **Token = a name, not a mechanism** — operator-chosen (`nvidia-gpu`), decoupled
  from how each side reaches it (VM hostdev vs pod `--device`); pure
  set-intersection unifies pod-vs-VM contention.
- **GPU auto-allocation** — when a token ALSO carries a `build.yml` `resource:`
  `gpu:` selector (`resource: {nvidia-gpu: {gpu: {vendor: "0x10de"}}}`), a
  `target: vm` claimant's `charly vm create` auto-detects the matching GPU,
  persists its `<hostdev>` into the per-host `instance.yml`, and injects it — or
  FAILS HARD when no matching card exists. Operator-authored hostdevs win (no
  double-inject). See `/charly-internals:disposable` "resource-arbitration axis"
  + `/charly-build:build` `resource:`.
- **`charly preempt status`** lists active leases + flags STRANDED ones (claimant
  gone). **`charly preempt restore [claimant]`** reconciles stranded leases (also run
  automatically at the next acquire) / force-releases a named one. A holder is
  NEVER left permanently stopped — the ledger
  (`~/.local/share/charly/preemption/leases.yml`) is written before any stop, and
  restore = "start every listed holder that isn't running".
- **Orthogonal to disposable/ephemeral** — no derivation either way; a deploy may
  be both preemptible and disposable. Full reference: `/charly-internals:disposable`
  "The resource-arbitration axis".

## Sibling peers (`peer:`) — companion deployments brought up alongside

`peer:` on a `DeploymentNode` declares **companion deployments brought up
ALONGSIDE** it on the shared `charly` network — *siblings*, not children. Contrast
`nested:`, whose children run **inside** this node's venue and are addressed by a
dotted path (`parent.child`). A peer is a companion **instrument**: the canonical
case is a Chrome DRIVER pod that CDP-probes a SEPARATE web-server SUBJECT, where a
check on the subject carries `on: <peer>` (see `/charly-eval:eval` "Cross-deployment
probing"). The SAME field + lifecycle serve a `kind: eval` bed and a `kind: deploy`
operator deployment — one codebase.

```yaml
deploy:
  webapp:                          # an operator deployment + a companion
    target: pod
    box: web
    peer:
      chrome:                      # a SIBLING brought up on the shared charly net
        target: pod
        box: chrome-headless       # a full DeploymentNode (its own target/box/port/…)
        port: [auto]
```

- **Folded to addressable top-level entries.** At load time `foldPeers` registers
  each peer as a top-level Deploy entry (so `charly config <peer>` / `charly start <peer>`
  resolve it through the exact path any deploy uses) carrying a derived `PeerOf:
  <owner>`. A peer name must be **globally unique** (a collision with any
  deploy/bed/peer is a hard load error) and carry **no `.`** (same rules as
  `nested:` keys + bed names); the author keeps peer **host ports disjoint** (the
  loader does not check ports — `[auto]` avoids fixed-port collisions).
- **One shared lifecycle (R3).** `bringUpPeers` / `tearDownPeers` (`charly/deploy_peers.go`)
  bring peers up after the owner and tear them down with it, by shelling out to the
  SAME verbs — a pod peer via `charly config` + `charly start` (+ readiness wait), a
  non-pod peer via `charly deploy add` / `charly deploy del`. Wired into `charly deploy add` /
  `charly deploy del` (operator path) AND the `kind: eval` bed runner (the bed's
  `--node-only` add never double-deploys; the runner brings peers up after the root
  starts). A bed's `charly update` (destroy + rebuild) tears peers down and back up too.
- **Disposability is inherited, never invented.** `foldPeers` promotes a peer to
  `disposable: true` only when its OWNER is disposable (so a disposable bed's
  rebuild is authorized to tear the peer down); a peer of a non-disposable operator
  deploy stays non-disposable. No new autonomy is granted — peers are components of
  their owner, touched only by the owner's explicit add/del/update (R6,
  `/charly-internals:disposable`).
- **Peers are NOT eval-live'd.** A `kind: eval` bed evaluates its SUBJECT (root +
  any `nested:` children via `bedEvalLiveRefs`); peers are instruments, never
  evaluated themselves — the subject's `on: <peer>` checks drive *through* them.
- **Addressing the subject from a driven probe** uses the `${PEER_HOST:<name>}` /
  `${PEER_ENDPOINT:<name>:<port>}` variables — see `/charly-eval:eval` "Cross-deployment
  probing".

## Cross-References

**Deploy surface:**
- `/charly-local:local-deploy` — Local-target execution model: LocalDeployTarget, ledger, gates, 15 ReverseOp kinds, sudo batching
- `/charly-internals:install-plan` — The InstallPlan IR shared by `charly box build` (OCITarget), pod deploys (PodDeployTarget), and local deploys (LocalDeployTarget)
- `/charly-internals:local-infra` — Supporting Go files for local deploys: hostdistro, ledger, builder_run, shell_profile, reverse_ops, service_render, deploy_ref

**Deploy-adjacent commands:**
- `/charly-build:pull` — Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path
- `/charly-automation:sidecar` — Sidecar containers, pod networking, Tailscale exit nodes, Environment Contract (provides filtering)
- `/charly-core:service` — Service lifecycle (start/stop/update/remove)
- `/charly-core:start` — Ergonomic alias for `charly deploy add <image> <image>` (container target)
- `/charly-core:stop` — Ergonomic alias for `charly deploy del <name>`
- `/charly-core:charly-update` — Per-instance update pattern; equivalent to `charly deploy add <name> --pull`
- `/charly-core:charly-config` — Resource cap flags (`--memory-max/high/swap/cpus`), provides filtering, env_require enforcement, NO_PROXY auto-enrichment, `--sidecar`, `-i` instance support, MCP name disambiguation
- `/charly-automation:enc` — Encrypted storage commands (charly config mount/unmount)
- `/charly-eval:vnc` — VNC password setup for desktop containers
- `/charly-vm:vm` — Virtual machine deployment (charly vm)
- `/charly-build:build` — Building images before deployment (+ the `--no-cache` intermediate scratch-stage caveat)
- `/charly-build:charly-mcp-cmd` — verify the MCP endpoints declared by `provides.mcp:` entries are actually reachable (`charly eval mcp ping <image>`); note the **port-publishing gotcha** when a `port:` override in deploy.yml predates a newly-added mcp-providing layer
- `/charly-image:image` — Image configuration, OCI label emission, `labels.go:238` tunnel read-skip
- `/charly-image:layer` — Unified `service:` schema (use_packaged + structured custom), `env_provide`/`env_require`/`env_accept` field declarations, security resource caps
- `/charly-eval:eval` — Local `eval:` in deploy.yml overlays image-baked deploy defaults: entries with matching `id:` replace, otherwise append. `id: X, skip: true` disables a baked check without a replacement.

**Canonical layer worked examples:**
- `/charly-selkies:chrome` — cgroup resource-caps consumer (caps bound a Chrome crash loop)
- `/charly-infrastructure:supervisord` — Event listener pattern triggered by the caps; ServiceSchemaDef that renders `service:` entries to supervisord INI
- `/charly-infrastructure:postgresql` — Canonical `use_packaged:` entry (packaged unit reuse)
- `/charly-ollama:ollama`, `/charly-hermes:hermes` — Custom `service:` entries
- `/charly-selkies:selkies-labwc` — Multi-instance proxy deployment, tunnel inheritance workaround

## Cross-kind name reuse + ResolveDeployRef precedence

A deploy entry's key in `deploy:` lives in its own namespace. The same name MAY simultaneously be a candy, a `box:` entry, a `pod:` entry, a `vm:` entry, a `k8s:` entry, a `local:` entry — and the deploy entry's cross-reference fields (`box:`, `vm:`, `local:`, `k8s:`) are scoped to the matching kind, no fall-through. Concrete worked example: this repo's `deploy.charly-cachyos` references `local.charly-cachyos` via `local: charly-cachyos` — same name across two namespaces.

`ResolveDeployRef` (used by `charly deploy add <name> <ref>`): when a name exists as BOTH a box and a candy, box-first precedence wins for the primary `<ref>` positional. The `--add-candy <ref>` path goes through `ResolveDeployRefAsLayer` which is candy-first. Same-name box and candy is permitted.

The loader raises a hard load-time error on the obsolete `deploy.qc` / `deploy.cachyos-dx` keys, and on the obsolete `kind: deployment` doc / root-key `deployment:` (the deploy kind is `kind: deploy`); every such error points at `charly migrate`. See `/charly-build:migrate`.

## When to Use This Skill

**MUST be invoked** when the task involves quadlet generation, tunnels, bind mounts, or deploy overlays. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** After `/charly-build:build`, before `/charly-core:service`.
Previous step: `/charly-build:build` (build the image). Next step: `/charly-core:service` (start, status, logs).

## Live-deploy verification is mandatory (see `/charly-eval:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/charly-internals:disposable`). Use `charly update <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `charly deploy add <name> <ref> --disposable` or mark a VM in vm.yml.

**After committing the source-level fix, `charly update` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
