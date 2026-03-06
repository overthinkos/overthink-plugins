---
name: deploy
description: |
  Deployment: quadlet systemd services, bootc disk images (QCOW2/RAW),
  VM management, registry push, tunnels, encrypted storage, and deploy overlays.
  Use for production deployment workflows.
---

# Deploy - Deployment

## Overview

Overthink supports multiple deployment modes: quadlet systemd user services for persistent containers, bootc disk images for bare-metal/VM installs, registry push for CI/CD, tunnel configuration for external access, and encrypted bind mounts for sensitive data.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Enable quadlet | `ov enable <image>` | Generate .container file, daemon-reload |
| Disable quadlet | `ov disable <image>` | Disable auto-start |
| Build QCOW2 | `ov vm build <image>` | Build QCOW2 disk image |
| Build RAW | `ov vm build <image> --type raw` | Build RAW disk image |
| Create VM | `ov vm create <image>` | Create VM from disk image |
| Start VM | `ov vm start <image>` | Start a VM |
| Stop VM | `ov vm stop <image>` | Stop a VM |
| Destroy VM | `ov vm destroy <image>` | Remove VM definition |
| List VMs | `ov vm list` | List VMs and status |
| VM console | `ov vm console <image>` | Attach to serial console |
| SSH into VM | `ov vm ssh <image>` | SSH into a VM |
| Push to registry | `ov build --push` | Multi-platform push |
| Seed bind mounts | `ov seed <image>` | Copy image data to empty bind mount dirs |
| Init encryption | `ov crypto init <image>` | Initialize gocryptfs dirs |
| Mount encrypted | `ov crypto mount <image>` | Mount encrypted volumes |
| Unmount encrypted | `ov crypto unmount <image>` | Unmount encrypted volumes |
| Change password | `ov crypto passwd <image>` | Change encryption password |

## Quadlet Services

User-level systemd services via podman quadlet. No root required.

### Setup

```bash
ov config set run_mode quadlet
ov config set engine.run podman
loginctl enable-linger $USER    # Required for user services
```

### Workflow

```bash
ov enable my-app -w ~/project    # Generate .container file, transfer image
ov enable my-app -i prod -w ~/prod -e ENV=production  # Named instance with env
ov start my-app                  # systemctl --user start
ov status my-app                 # systemctl --user status
ov logs my-app -f                # journalctl --user -u (follow)
ov update my-app                 # Re-transfer image, restart
ov stop my-app                   # systemctl --user stop
ov disable my-app                # Disable auto-start
ov remove my-app                 # Stop + remove .container file
ov remove my-app --volumes       # Also remove named volumes
ov remove my-app -e KEY=VALUE    # Set env vars for lifecycle hooks
```

With `auto_enable=true`, `ov start` auto-generates the quadlet file on first run.

Generated files: `~/.config/containers/systemd/ov-<image>.container` (or `ov-<image>-<instance>.container` with `-i`). Service name: `ov-<image>.service`. Container name: `ov-<image>`. Ports bound to configured `bind_address`. Entrypoint: `supervisord -n -c /etc/supervisord.conf`. Auto-restart on failure via `WantedBy=default.target`.

### Environment Variables in Quadlet

Quadlet handles env vars via two mechanisms:

- **`Environment=`** lines for CLI `-e` flags (inline in .container file)
- **`EnvironmentFile=`** directive for file-sourced vars (`--env-file`, workspace `.env`, or `env_file` in images.yml)

When `EnvironmentFile=` is used, only explicit CLI `-e` vars appear as inline `Environment=` to avoid duplication.

### Security in Quadlet

Layer and image-level security settings are applied as `PodmanArgs=` in the quadlet file:

- `privileged: true` → `PodmanArgs=--privileged`
- `cap_add` → `PodmanArgs=--cap-add=<CAP>`
- `devices` → `PodmanArgs=--device=<DEV>`
- `security_opt` → `PodmanArgs=--security-opt=<OPT>`

Source: `ov/security.go` (`CollectSecurity`, `SecurityArgs`).

### Lifecycle Hooks

Layers can declare lifecycle hooks that run on the host at specific points:

```yaml
# In layer.yml:
hooks:
  post_enable: |
    echo "Service enabled for $OV_IMAGE"
  pre_remove: |
    echo "Cleaning up before removal"
```

- `post_enable` runs after `ov enable` generates the quadlet and reloads systemd
- `pre_remove` runs before `ov remove` stops and removes the service

Hooks from multiple layers are concatenated in layer order. Use `ov remove -e KEY=VALUE` to pass environment variables to hook scripts.

Source: `ov/hooks.go` (`CollectHooks`, `RunHook`).

### Image Transfer

When `engine.build=docker`, `ov enable` auto-detects if the image is missing from podman and transfers via `docker save | podman load`. `ov update` re-transfers if needed and restarts the service.

Source: `ov/quadlet.go` (generation), `ov/commands.go` (command structs).

## Bootc Disk Images & VM Management

For images with `bootc: true` in images.yml. Uses `bootc install to-disk` inside a privileged container to write a disk image. Requires a container engine (podman or docker).

### Building Disk Images

```bash
ov vm build my-bootc-image                  # Build QCOW2 (default)
ov vm build my-bootc-image --type raw       # Build RAW disk
ov vm build my-bootc-image --size 20G       # Custom disk size
ov vm build my-bootc-image --root-size 10G  # Custom root partition
ov vm build my-bootc-image --console        # Enable console output for debugging
```

Output goes to `output/<type>/disk.<type>`. ISO format is not supported -- use qcow2 or raw.

### VM Lifecycle

```bash
ov vm create my-image                  # Create VM from built disk image
ov vm create my-image --ram 8G --cpus 4  # Custom resources
ov vm create my-image --ssh-key auto   # Auto-detect SSH key (default)
ov vm create my-image --ssh-key generate  # Generate new SSH keypair
ov vm create my-image -i prod          # Named instance
ov vm start my-image                   # Start a stopped VM
ov vm stop my-image                    # Graceful shutdown
ov vm stop my-image --force            # Force stop (destroy)
ov vm destroy my-image                 # Remove VM definition
ov vm destroy my-image --disk          # Also delete disk image
ov vm list                             # List running VMs
ov vm list -a                          # Include stopped VMs
ov vm console my-image                 # Attach to serial console
ov vm ssh my-image                     # SSH into VM
ov vm ssh my-image -p 2222 -l user     # Custom port and user
ov vm ssh my-image -- ls /tmp          # Run command via SSH
```

VM name convention: `ov-<image>[-<instance>]`. All VMs use session-level libvirt URI (`qemu:///session`).

### VM Backends

Backend resolution order: **libvirt -> qemu** (auto-detected). Libvirt is detected by checking for the session socket; QEMU by checking for `qemu-system-*` binary.

| Backend | Requires | Notes |
|---------|----------|-------|
| `libvirt` | libvirt session daemon (socket) | Default, recommended |
| `qemu` | `qemu-system-*` | Direct QEMU, state in `~/.local/share/ov/vm/` |

SSH keys are injected at VM creation time via SMBIOS credentials (systemd-based). The `--ssh-key` flag controls key injection: `auto` (default, uses `~/.ssh/*.pub`), `generate` (creates new keypair), `none` (no key injection), or a path to a `.pub` file.

Source: `ov/smbios_credentials.go`.

Override: `ov config set vm.backend libvirt`

### VM Configuration

Per-image `vm:` section in images.yml:

```yaml
images:
  my-bootc-image:
    bootc: true
    base: "quay.io/fedora/fedora-bootc:43"
    layers: [sshd, qemu-guest-agent, bootc-config]
    vm:
      disk_size: "10 GiB"
      root_size: "10G"
      ram: "4G"
      cpus: 2
      rootfs: ext4            # ext4, xfs, or btrfs
      kernel_args: "console=ttyS0,115200n8"
      ssh_port: 2222
      transport: registry     # registry, containers-storage, oci, oci-archive
      firmware: ""            # uefi-secure, uefi-insecure, bios
      network: ""             # user, or bridge name
```

User-level config via `ov config set`:

| Key | Env Var | Default |
|-----|---------|---------|
| `vm.backend` | `OV_VM_BACKEND` | `auto` |
| `vm.disk_size` | `OV_VM_DISK_SIZE` | `10 GiB` |
| `vm.root_size` | `OV_VM_ROOT_SIZE` | (empty) |
| `vm.ram` | `OV_VM_RAM` | `4G` |
| `vm.cpus` | `OV_VM_CPUS` | `2` |
| `vm.rootfs` | `OV_VM_ROOTFS` | `ext4` |
| `vm.transport` | `OV_VM_TRANSPORT` | (empty) |

Resolution chain: **image-level vm: -> defaults vm: -> ov config -> hardcoded defaults**.

### Libvirt XML Injection

Layers and images can declare raw libvirt XML snippets via the `libvirt` field. These are injected into the VM domain XML after creation.

```yaml
# In layer.yml (e.g., qemu-guest-agent):
libvirt:
  - <channel type='unix'><target type='virtio' name='org.qemu.guest_agent.0'/></channel>

# In images.yml:
images:
  my-image:
    libvirt:
      - "<filesystem type='mount'>...</filesystem>"
```

Device elements (channel, disk, interface, etc.) are placed inside `<devices>`. Other elements are placed before `</domain>`. Snippets are deduplicated by exact string match.

Source: `ov/vm.go`, `ov/vm_build.go`, `ov/libvirt.go`.

## Tunnel Configuration

Expose services outside the container host via tunnels.

### Tailscale Serve (tailnet-private, default)

Exposes a port to your Tailscale network only. No FQDN needed -- Tailscale handles TLS automatically.

```yaml
tunnel: tailscale
# or expanded:
tunnel:
  provider: tailscale
  port: 2283
  https: 443          # default: 443
```

Allowed HTTPS ports for serve: 80, 443, 3000-10000, 4443, 5432, 6443, 8443.

### Tailscale Funnel (public internet)

Exposes a port to the public internet via Tailscale's edge network.

```yaml
tunnel:
  provider: tailscale
  funnel: true
  port: 8080
  https: 443    # must be 443, 8443, or 10000
```

### Cloudflare Tunnel

Routes traffic through Cloudflare's network. Requires `fqdn`.

```yaml
tunnel:
  provider: cloudflare
  port: 3001
  tunnel: my-tunnel    # optional, defaults to ov-<image>
fqdn: "app.example.com"
```

### Resolution

`tunnel` inherits from defaults (image -> defaults -> nil). The `port` field defaults to the first route port from layers if not specified. For tailscale, `https` defaults to 443.

Source: `ov/tunnel.go`, `ov/validate.go` (`validateTunnel`), `ov/quadlet.go` (systemd integration).

## Deploy Overlay

Per-machine deployment overrides in `~/.config/ov/deploy.yml` (not checked into git):

```yaml
images:
  my-app:
    tunnel:
      provider: cloudflare
      port: 2283
    fqdn: "app.example.com"
    bind_mounts:
      - name: secrets
        path: "~/.myapp/secrets"
        encrypted: true
    ports:
      - "2283:2283"
```

Only deployment/runtime fields allowed: `tunnel`, `fqdn`, `acme_email`, `bind_mounts`, `ports`, `env`, `env_file`.

## Bind Mounts

Image-level host-path bind mounts declared in `images.yml` or `deploy.yml`. Two modes: plain (direct bind) and encrypted (gocryptfs-managed). Not inherited from defaults -- deployment-specific.

### Configuration

```yaml
bind_mounts:
  - name: data
    host: "~/data/myapp"        # host dir, required for plain mounts
    path: "~/.myapp"            # container path, ~ expanded to resolved home

  - name: secrets
    path: "~/.myapp/secrets"    # container path
    encrypted: true             # gocryptfs, host managed by ov
```

**Rules:**
- `encrypted: false` (default): `host` is required -- direct bind mount
- `encrypted: true`: `host` is forbidden -- ov manages cipher/plain dirs at `encrypted_storage_path`
- `path`: container mount path (required). `~`/`$HOME` expanded to the image's resolved home dir
- `host`: host mount path (plain mounts only). `~`/`$HOME` expanded to the user's actual home
- `name`: unique identifier, must match `^[a-z0-9]+(-[a-z0-9]+)*$`
- Bind mount names must not collide with layer volume names (same namespace)

### Bind Mount / Volume Override

When a bind mount has the same name as a layer volume, the bind mount **overrides** the volume. The named volume is not created -- the bind mount is used instead. This is an informational note on stderr. This allows deploy.yml to replace layer-declared volumes with encrypted bind mounts.

### Encrypted Bind Mounts

```bash
ov crypto init my-app [--volume NAME]     # Initialize gocryptfs cipher dirs
ov crypto mount my-app [--volume NAME]    # Mount (prompts for password once)
ov crypto unmount my-app [--volume NAME]  # Unmount
ov crypto status my-app                   # Show status
```

Storage layout: `~/.local/share/ov/encrypted/ov-<image>-<name>/cipher/` and `plain/`. Override path: `OV_ENCRYPTED_STORAGE_PATH`.

### Single Password

When an image has multiple encrypted bind mounts, `ov crypto init`, `ov crypto mount`, and the generated crypto systemd unit all use `systemd-ask-password --id=ov-<image>` to cache the passphrase in the kernel keyring. Prompted once and reused for all volumes.

### Integration

- **`ov seed <image>`**: copies default data from image into empty bind mount directories before first start
- **`ov shell`/`ov start` (direct)**: resolves bind mounts, verifies plain dirs exist and encrypted volumes are mounted, appends `-v <host>:<container>` flags
- **`ov enable` (quadlet)**: plain mounts as `Volume=` lines; encrypted mounts generate a companion `ov-<image>-crypto.service` with `Requires=`/`After=`
- **`ov remove`**: removes companion crypto service file. `--volumes` also removes named volumes. Runs `pre_remove` hooks before cleanup
- **`ov inspect --format bind_mounts`**: outputs `NAME\tHOST\tPATH\tENCRYPTED`
- **`ov crypto passwd <image>`**: changes the gocryptfs password for all encrypted volumes of an image

Source: `ov/crypto.go`, `ov/validate.go` (`validateBindMounts`).

## Cross-References

- `/overthink:build` -- Building images before deployment
- `/overthink:run` -- Runtime commands (start, stop, shell)
- `/overthink:image` -- Image configuration

## When to Use This Skill

Use when the user asks about:

- Quadlet/systemd service deployment
- Bootc disk images (QCOW2, RAW)
- VM management (create, start, stop, destroy, console, ssh)
- Pushing images to registries
- Tunnel configuration (Tailscale, Cloudflare)
- Encrypted storage and bind mounts
- Deploy overlays
- "How do I deploy X to production?"
