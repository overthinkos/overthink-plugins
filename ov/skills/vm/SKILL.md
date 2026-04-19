---
name: vm
description: |
  MUST be invoked before any work involving: virtual machines, ov vm commands, bootc disk images, libvirt/QEMU backends, or VM lifecycle management.
---

# VM - Virtual Machine Management

## Overview

`ov vm` commands build disk images from bootc container images and manage virtual machines via libvirt or QEMU. Images must have `bootc: true` in image.yml. Disk images are built using `bootc install to-disk` inside a privileged container.

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

For images with `bootc: true` in image.yml. Uses `bootc install to-disk` inside a privileged container.

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

Override: `ov settings set vm.backend libvirt`

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

### Per-Image in image.yml

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

Set via `ov settings set`:

| Key | Env Var | Default |
|-----|---------|---------|
| `vm.backend` | `OV_VM_BACKEND` | `auto` |
| `vm.disk_size` | `OV_VM_DISK_SIZE` | `10 GiB` |
| `vm.root_size` | `OV_VM_ROOT_SIZE` | (empty) |
| `vm.ram` | `OV_VM_RAM` | `4G` |
| `vm.cpus` | `OV_VM_CPUS` | `2` |
| `vm.rootfs` | `OV_VM_ROOTFS` | `ext4` |
| `vm.transport` | `OV_VM_TRANSPORT` | (empty) |

Resolution chain: **image-level vm: -> defaults vm: -> ov settings -> hardcoded defaults**.

## Libvirt XML Injection

Layers and images can declare raw libvirt XML snippets via the `libvirt` field. These are injected into the VM domain XML after creation.

```yaml
# In layer.yml (e.g., qemu-guest-agent):
libvirt:
  - <channel type='unix'><target type='virtio' name='org.qemu.guest_agent.0'/></channel>

# In image.yml:
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
ov image build my-bootc-image
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

## Known bootc-VM caveats

Covers non-obvious issues that only surface when a bootc image actually boots under QEMU. The canonical end-to-end worked example is `/ov-images:selkies-desktop-bootc`.

### Privileged container needs `-v /dev:/dev`

`ov vm build` runs `bootc install to-disk --via-loopback` inside a privileged container (via `sudo podman` under `engine.rootful=sudo`, or podman-machine under `engine.rootful=machine`). `--privileged` alone is not enough — the container's `/dev` is a tmpfs in its own mount namespace, so `losetup`'s `LOOP_CTL_GET_FREE` can create a `/dev/loopN` in the kernel but the node is invisible inside the container. `ov/vm_build.go` adds `-v /dev:/dev` so the container shares the host's `/dev`. Symptom without it: `losetup: <path>: failed to set up loop device: No such file or directory`.

### `vm.ssh_port` is honoured end-to-end

The QEMU hostfwd SSH mapping (`hostfwd=tcp::<ssh_port>-:22`) and the libvirt `<portForward>` range both read `vm.ssh_port` from the image's Vm OCI label. Plumbing lives at `ov/vm.go` (`createQemu`/`createLibvirt`) and `ov/vm_libvirt.go` (`buildDomainXML`). Older `ov` binaries hardcoded 2222; if your VM creates with SSH on 2222 despite `vm.ssh_port: 2250` in `image.yml`, the binary is stale — run `task build:ov`. Also: `image.yml` `ports:` flow straight into QEMU hostfwd, so a `"2222:2222"` publish will collide with `vm.ssh_port: 2222` — don't declare both.

### Rootful ↔ rootless podman storage split

Under `engine.rootful=sudo`, `ov vm build` invokes `sudo podman`, which reads from root's container storage (`/var/lib/containers/storage`), not your rootless storage. After every `ov image build`, refresh rootful storage:

```bash
podman save ghcr.io/<registry>/<image>:latest -o /tmp/img.tar
sudo podman load -i /tmp/img.tar && rm -f /tmp/img.tar
```

Symptom without refresh: `Trying to pull ... 403 Forbidden` (bootc falls back to registry).

### `engine.rootful=machine` path (rootless containers, `/ov-images:selkies-desktop-ov`)

When `ov vm build` is invoked from inside a rootless container (no host `sudo` available), the engine auto-picks `engine.rootful=machine`. That **spawns a podman-machine VM** (nested VM #1, KVM-accelerated via `/dev/kvm` passthrough), mounts the calling user's home into the machine, and runs `bootc install to-disk --via-loopback` inside it. The machine has its **own rootful storage** (`podman --connection ov-root`), separate from the calling container's nested rootless store — so the image ref passed to `ov vm build` must be resolvable from the machine's storage, not just the calling container's.

Symptom: `Trying to pull ghcr.io/.../<image>:latest ... 403 Forbidden` **inside** the `podman --connection ov-root` step, even after the image is loaded in the calling container's nested rootless store.

Cross-storage load pattern (run inside the calling container):

```bash
# Assume the image is already in the calling container's nested rootless store
# (via `podman load -i` of a host-side `podman save` tarball).

podman save -o /tmp/bootc.tar ghcr.io/<registry>/<image>:latest
podman --connection ov-root load -i /tmp/bootc.tar
rm /tmp/bootc.tar
ov vm build <image> --transport containers-storage
```

`--transport containers-storage` forces `bootc install` to pull from the machine's local store instead of the registry. Worked example + two-level nesting end-to-end run lives in `/ov-images:selkies-desktop-ov` ("Two-level nested-virtualization proof").

### `ov vm stop` state-sync race under the QEMU backend

Observed 2026-04-19 on `selkies-desktop-bootc` inside `selkies-desktop-ov`: `ov vm stop <image>` returns `Stopped VM ov-<image>` and exits 0, but a subsequent `ov vm list -a` still reports the VM as `running` until `ov vm destroy` is called — at which point list goes empty cleanly. The libvirt backend is unaffected; this is a QEMU-backend state-sync bug in `ov`. Workaround: proceed straight to `ov vm destroy` rather than treating the residual "running" as a stop failure. Tracked as a follow-up ov bug; no source change in this skill's scope.

### Host kernel/module-dir mismatch

On rolling distros (Arch, Fedora) after a kernel upgrade, `/lib/modules/$(uname -r)` may be missing while `/lib/modules/<new-kernel>/` is present. `modprobe loop` fails with `Module loop not found in directory /lib/modules/$(uname -r)`, losetup cannot create a loop device, and the VM disk build hangs at `losetup: failed to set up loop device`. Resolution: **reboot**. Quick check:

```bash
ls /lib/modules/$(uname -r) >/dev/null 2>&1 && echo "ok" || echo "reboot needed"
```

### Disk exhaustion from `output/`

`bootc install to-disk` writes a ~40 GiB sparse raw file AND a qcow2. A failed `qemu-img convert` (disk-full, corrupted cache) leaves both behind. Clean `output/raw/disk.raw` after successful qcow2 conversion, and keep a close eye on `df -h /` before a fresh VM build.

### `qemu-guest-agent` enabled/inactive under QEMU user-net

Expected. The agent needs a `virtio-serial` channel that ov's QEMU backend doesn't expose under user-net. Switch to the libvirt backend (`ov settings set vm.backend libvirt`) to activate it.

## Cross-References

- `/ov:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov:build` -- Building container images before VM disk builds; also covers the stale-ov-binary and buildah-cache-mount caveats
- `/ov:config` -- VM default settings (`vm.*` config keys)
- `/ov:image` -- `bootc`, `vm:`, and `libvirt` fields in image.yml; also the external-base `distro:` requirement
- `/ov:layer` -- `libvirt` field in layer.yml
- `/ov-images:selkies-desktop-bootc` -- Canonical end-to-end bootc worked example (port remap, known caveats, verification recipes)
- `/ov-layers:bootc-config` -- Bootc-side boot wiring (tty1 autologin, graphical target, systemd-user supervisord)

## When to Use This Skill

**MUST be invoked** when the task involves virtual machines, ov vm commands, bootc disk images, libvirt/QEMU backends, or VM lifecycle management. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Standalone workflow. VM management is separate from container lifecycle.
