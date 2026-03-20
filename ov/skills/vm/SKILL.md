---
name: vm
description: |
  MUST be invoked before any work involving: virtual machines, ov vm commands, bootc disk images, libvirt/QEMU backends, or VM lifecycle management.
---

# VM - Virtual Machine Management

## Overview

`ov vm` commands build disk images from bootc container images and manage virtual machines via libvirt or QEMU. Images must have `bootc: true` in images.yml. Disk images are built using `bootc install to-disk` inside a privileged container.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Build QCOW2 | `ov vm build <image>` | Build QCOW2 disk image (default) |
| Build RAW | `ov vm build <image> --type raw` | Build RAW disk image |
| Create VM | `ov vm create <image>` | Create VM from built disk image |
| Start VM | `ov vm start <image>` | Start a stopped VM |
| Stop VM | `ov vm stop <image>` | Graceful shutdown |
| Force stop | `ov vm stop <image> --force` | Force stop (destroy) |
| Destroy VM | `ov vm destroy <image>` | Remove VM definition |
| Destroy + disk | `ov vm destroy <image> --disk` | Remove VM and delete disk image |
| List VMs | `ov vm list` | List running VMs |
| List all | `ov vm list -a` | Include stopped VMs |
| Console | `ov vm console <image>` | Attach to serial console |
| SSH | `ov vm ssh <image>` | SSH into VM |

## Building Disk Images

For images with `bootc: true` in images.yml. Uses `bootc install to-disk` inside a privileged container.

```bash
ov vm build my-bootc-image                  # Build QCOW2 (default)
ov vm build my-bootc-image --type raw       # Build RAW disk
ov vm build my-bootc-image --size 20G       # Custom disk size
ov vm build my-bootc-image --root-size 10G  # Custom root partition
ov vm build my-bootc-image --console        # Enable console output for debugging
```

Output goes to `output/<type>/disk.<type>`. ISO format is not supported -- use qcow2 or raw.

The `--transport` flag controls how the container image is passed to bootc: `registry` (pull from registry), `containers-storage` (local podman storage), `oci` (OCI directory), `oci-archive` (OCI tarball).

Source: `ov/vm_build.go`.

## VM Lifecycle

```bash
ov vm create my-image                       # Create from disk image
ov vm create my-image --ram 8G --cpus 4     # Custom resources
ov vm create my-image --ssh-key auto        # Auto-detect SSH key (default)
ov vm create my-image --ssh-key generate    # Generate new SSH keypair
ov vm create my-image --ssh-key none        # No key injection
ov vm create my-image --ssh-key ~/.ssh/id_ed25519.pub  # Specific key
ov vm create my-image -i prod               # Named instance
ov vm start my-image                        # Start a stopped VM
ov vm stop my-image                         # Graceful shutdown
ov vm stop my-image --force                 # Force stop (destroy)
ov vm destroy my-image                      # Remove VM definition
ov vm destroy my-image --disk               # Also delete disk image
ov vm list                                  # List running VMs
ov vm list -a                               # Include stopped VMs
ov vm console my-image                      # Attach to serial console
ov vm ssh my-image                          # SSH into VM
ov vm ssh my-image -p 2222 -l user          # Custom port and user
ov vm ssh my-image -- ls /tmp               # Run command via SSH
```

VM name convention: `ov-<image>[-<instance>]`. All VMs use session-level libvirt URI (`qemu:///session`).

## VM Backends

Backend resolution order: **libvirt -> qemu** (auto-detected). Libvirt is detected by checking for the session socket; QEMU by checking for `qemu-system-*` binary.

| Backend | Requires | Notes |
|---------|----------|-------|
| `libvirt` | libvirt session daemon (socket) | Default, recommended |
| `qemu` | `qemu-system-*` | Direct QEMU, state in `~/.local/share/ov/vm/` |

Override: `ov config set vm.backend libvirt`

Source: `ov/vm.go`, `ov/vm_libvirt.go`, `ov/vm_qemu.go`.

## SSH Key Injection

SSH keys are injected at VM creation time via SMBIOS credentials (systemd-based). The `--ssh-key` flag controls key injection:

| Value | Behavior |
|-------|----------|
| `auto` (default) | Uses first `~/.ssh/*.pub` found |
| `generate` | Creates new keypair in `~/.local/share/ov/vm/ssh/` |
| `none` | No key injection |
| `/path/to/key.pub` | Uses the specified public key |

Source: `ov/smbios_credentials.go`.

## VM Configuration

### Per-Image in images.yml

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

### User-Level Defaults

Set via `ov config set`:

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

## Libvirt XML Injection

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

Source: `ov/libvirt.go`.

## Common Workflows

### Build and Run a VM

```bash
ov build my-bootc-image
ov vm build my-bootc-image
ov vm create my-bootc-image --ram 8G --cpus 4
ov vm start my-bootc-image
ov vm ssh my-bootc-image
```

### Debug VM Boot Issues

```bash
ov vm build my-bootc-image --console     # Enable console output
ov vm create my-bootc-image
ov vm start my-bootc-image
ov vm console my-bootc-image             # Watch boot messages
```

### Multiple VM Instances

```bash
ov vm create my-image -i dev
ov vm create my-image -i staging
ov vm start my-image -i dev
ov vm ssh my-image -i dev
```

## Cross-References

- `/ov:build` -- Building container images before VM disk builds
- `/ov:config` -- VM default settings (`vm.*` config keys)
- `/ov:image` -- `bootc`, `vm:`, and `libvirt` fields in images.yml
- `/ov:layer` -- `libvirt` field in layer.yml

## When to Use This Skill

**MUST be invoked** when the task involves virtual machines, ov vm commands, bootc disk images, libvirt/QEMU backends, or VM lifecycle management. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Standalone workflow. VM management is separate from container lifecycle.
