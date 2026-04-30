---
name: arch
description: |
  First kind:vm entity with source.kind: cloud_image — fetches the Arch Linux cloud
  qcow2 from pkgbuild.com, applies cloud-init, boots under libvirt/QEMU via BIOS
  firmware + virtio-gpu. Documents the stale-BOOTX64.EFI RCA, the
  simpledrm→qxldrmfb takeover race, the adopt-user pattern, and resource sizing.
  MUST be invoked before editing arch in vms.yml or authoring another
  cloud_image VM from a template.
---

# arch

Canonical `source.kind: cloud_image` VM in the repo. Boots an Arch Linux cloud image as a full VM with SSH + SPICE console access, cloud-init-provisioned SSH keys, virtio-gpu graphics, and the `ov` toolchain auto-installed inside the guest.

This skill is the **decision log** for every non-obvious choice in the entry — BIOS vs UEFI, virtio-gpu vs QXL, resource sizing vs package list pruning, adopt-user vs create-user. Every decision traces to a live-test RCA; the rationales belong here permanently so a future editor (human or agent) doesn't reopen the same questions.

## VM Configuration

| Setting | Value | Rationale |
|---|---|---|
| Source | `https://fastly.mirror.pkgbuild.com/images/latest/Arch-Linux-x86_64-cloudimg.qcow2` | Official Arch cloud image, CDN-fronted |
| Checksum type | `sha256` | Value empty → auto-resolve `<url>.SHA256` sidecar at fetch time |
| `base_user` | `arch` | Upstream account; adopt pattern below |
| Firmware | `bios` | Skip stale BOOTX64.EFI (Finding B) |
| Machine | `q35` | Modern PCIe chipset, works with both SeaBIOS and OVMF |
| Disk size | `40G` | Headroom for spice-vdagent (GTK3 + X11 ~1 GB installed) + layer overlays |
| RAM | `8G` | Avoid cloud-init stall during spice-vdagent install (Finding C) |
| CPUs | `4` | Parallel xz decompression + `pacman-key --populate` benefit from SMP |
| Network mode | `user` | QEMU user-mode with port forwards |
| SSH port | `2224` | Non-default to coexist with other dev VMs on 2222 |
| SSH key source | `generate` | Stable `~/.local/share/ov/vm/ov-arch/id_ed25519` across rebuilds |
| Video model | `virtio-gpu` | Modern default for Linux guests (Finding B, secondary) |
| SPICE listener | `type: socket` (UNIX, auto-path) | Enables zero-config remote GUI via `qemu+ssh://` (see "Connecting from a remote workstation" below). virt-manager and `remote-viewer` auto-forward UNIX sockets through libvirt RPC fd-passing; TCP-loopback listeners are never auto-tunneled. No TCP port bound. |
| `disposable` | `true` (LOAD-BEARING) | Authorizes `ov rebuild arch` to destroy + rebuild + restart unattended. This VM exists for live verification and rebuild-at-will. See `/ov-dev:disposable`. |
| `lifecycle` | `dev` (INFORMATIONAL) | Human-facing tier tag. Zero effect on disposability — the flag above is what matters. |

## Marked `disposable: true`

This is the repo's canonical verification target. It carries `disposable: true` in vms.yml, which means `ov rebuild arch` runs the destroy → build → create → start loop unattended — no user confirmation. The hook reminders in `.claude/hooks/` reference disposability specifically; this VM is what Claude is expected to verify against.

If you're implementing something that touches VM config, libvirt rendering, cloud-init, SPICE, or any VM-adjacent behavior, the expected verification loop is:

```bash
ov rebuild arch       # (destroy + build + create + start)
#  ... exploratory testing ...
# commit the source-level fix
ov rebuild arch       # fresh-rebuild re-verification (R10)
# paste BOTH outputs into the conversation
```

No other VM in this repo is disposable by default. To make another one rebuildable unattended, add `disposable: true` to its kind:vm entry (no derivation from `lifecycle:` — the flag is always explicit).

## Full VmSpec (from vms.yml)

```yaml
vms:
  arch:
    source:
      kind: cloud_image
      url: https://fastly.mirror.pkgbuild.com/images/latest/Arch-Linux-x86_64-cloudimg.qcow2
      checksum:
        type: sha256
      base_user: arch
    disk_size: 40G
    ram: 8G
    cpus: 4
    machine: q35
    firmware: bios
    network:
      mode: user
    ssh:
      port: 2224
      key_source: generate
    cloud_init:
      timezone: UTC
      packages:
        - sudo
        - spice-vdagent
      runcmd:
        - pacman-key --init
        - pacman-key --populate archlinux
        - systemctl enable --now spice-vdagentd
      ov_install:
        strategy: auto
    libvirt:
      devices:
        channels:
          - {type: spicevmc, name: com.redhat.spice.0}
        graphics:
          # Socket-only SPICE — virt-manager and `remote-viewer
          # --connect qemu+ssh://…` auto-forward the UNIX socket over
          # libvirt RPC, zero extra setup. libvirt auto-allocates the
          # socket path under $XDG_RUNTIME_DIR/libvirt/qemu/.
          - type: spice
            listen:
              - type: socket
        video:
          - {model: virtio, vram: 65536, heads: 1, accel3d: false}
        rng:
          - {model: virtio, backend: /dev/urandom}
        memballoon: {model: virtio}
```

## Connecting from a remote workstation

The VM's SPICE configuration above is specifically chosen so virt-manager on
another machine Just Works against a libvirt session here.

### virt-manager (zero ov involvement required)

```
virt-manager --connect qemu+ssh://o.atrawog.org/session
```

Double-click `ov-arch`. The SPICE console opens. Under the
hood, virt-manager (and the `virt-viewer --connect qemu+ssh://…
--attach <vm>` path it uses internally) reads the domain XML, sees
`<listen type='socket' socket='/…/spice.sock'/>`, and auto-tunnels by
spawning:

```
ssh <host> nc -U /<remote-socket-path>
```

stdio-piping SPICE traffic through the SSH control channel. **No `ssh
-L`, no ov commands, no password, no TCP port bound on the remote
host.**

**Required host dep**: `nc` (openbsd-netcat on Arch, netcat-openbsd on
Debian/Ubuntu, nmap-ncat on Fedora) must be installed on the libvirt
host. ov's `setup.sh` and `pkg/arch/PKGBUILD` install it automatically.
Without `nc`, virt-manager hangs at "Connecting to graphical console
for guest" — no error, just silent failure. Diagnose with
`ssh <host> which nc` (should return a path).

### `ov eval spice` with `--uri` (for CLI diagnostics / local artifacts)

To probe the remote VM from the CLI and write screenshots into the local
filesystem:

```bash
ov eval spice status arch --uri qemu+ssh://o.atrawog.org/session
ov eval spice screenshot arch --uri qemu+ssh://o.atrawog.org/session /tmp/shot.png
ov eval libvirt info arch --uri qemu+ssh://o.atrawog.org/session
```

`ov` opens an SSH connection, forwards the remote SPICE UNIX socket to a
local socket, and dials it — all transparent to the user. Set
`OV_LIBVIRT_URI=qemu+ssh://o.atrawog.org/session` to avoid repeating the flag.

### `ov --host o` (run ov on the remote machine)

For commands that don't need their output to land locally:

```bash
ov settings set hosts.o o.atrawog.org
ov --host o status
ov --host o vm list
ov --host o test spice status arch
ov --host o test spice screenshot arch - > /tmp/shot.png
```

`ov` re-execs itself over SSH (via your system's `ssh`, so `~/.ssh/config`
and agent forwarding just work). `-` as a screenshot path writes PNG bytes
to stdout so the pipeline composes naturally.

### `ov ssh tunnel` (for external clients like TigerVNC / bare remote-viewer)

```bash
ov ssh tunnel spice arch --uri qemu+ssh://o.atrawog.org/session
# prints: spice tunnel: spice+unix:///tmp/ov-tunnel-8e4c.sock
# Connect with: remote-viewer spice+unix:///tmp/ov-tunnel-8e4c.sock
```

Add `--tcp` to force a 127.0.0.1:<random> TCP forward for clients that
don't understand `spice+unix://`.


## Finding A — disk/FS resize works end-to-end

False alarm during live testing — verified at four layers:

| Layer | Observed |
|---|---|
| Host qcow2 overlay | `qemu-img info -U` → virtual size: 40 GiB |
| Guest kernel | `fdisk -l /dev/vda` → 40 GiB, 83886080 sectors |
| Guest partition | `/dev/vda3 … 39.7G Linux root (x86-64)` (grew from ~2G base) |
| Guest filesystem | `btrfs filesystem show` → size 39.70 GiB |

The cloud-utils-growpart + btrfs-online-resize pipeline that cloud-init runs at first boot works as designed. No workaround needed; the `ov vm build` qcow2 overlay resize at `disk_size:` suffices.

## Finding B — SPICE console blank: BIOS firmware bypasses the race

**Symptom observed under UEFI:** kernel dmesg shows a deferred fbcon takeover sequence:

```
[    0.555] [drm] Initialized simpledrm 1.0.0 for simple-framebuffer.0 on minor 0
[    0.560] fbcon: Deferring console take-over
[    2.412] qxl 0000:00:01.0: vgaarb: deactivate vga console
[    2.426] fbcon: qxldrmfb (fb0) is primary device
[    2.426] fbcon: Deferring console take-over
```

Agetty renders the login prompt to simpledrm's framebuffer at t~0.5s; qxldrmfb takes over at t=2.4s with a fresh (black) framebuffer, losing the prompt. `fbcon=nodefer` would suppress the deferral — and the Arch cloud image's `/etc/default/grub` already declares it. But `/proc/cmdline` on a UEFI-booted instance does NOT contain `fbcon=nodefer`.

**RCA of the missing kernel arg:**
- `/boot/grub/grub.cfg` on vda3 **does** contain `fbcon=nodefer` (2 occurrences).
- The EFI System Partition (vda2) holds **only** `/EFI/BOOT/BOOTX64.EFI`, no `grub.cfg`.
- `BOOTX64.EFI` is a PE32+ GRUB binary built via `grub-mkstandalone` with an **embedded grub.cfg** at image-build time. That embedded config predates the `fbcon=nodefer` addition; the standalone EFI binary was never regenerated when `/etc/default/grub` + `/boot/grub/grub.cfg` were updated.
- UEFI boot path: firmware → `BOOTX64.EFI` (stale embedded config, no `fbcon=nodefer`) → kernel. `/boot/grub/grub.cfg` on vda3 is never consulted.

**Fix: switch firmware to `bios`.** The Arch cloud image's GPT partition table preserves this option:

```
$ fdisk -l /dev/vda
Disklabel type: gpt
/dev/vda1    2048     4095     2048    1M BIOS boot        ← 1 MB BIOS boot partition
/dev/vda2    4096   618495   614400  300M EFI System
/dev/vda3  618496 41943006 41324511 19.7G Linux root
```

vda1 is a 1 MB BIOS boot partition — only needed for GRUB's core image under BIOS boot. Its existence proves the image was built to support both BIOS and EFI boot paths. With `firmware: bios`:

- QEMU boots vda1 via SeaBIOS → vda's MBR/GPT-protective-MBR → GRUB reads `/boot/grub/grub.cfg` from vda3 directly. **No ESP involvement. No stale BOOTX64.EFI issue.**
- No EFI framebuffer → **no simpledrm instantiated at boot.** Eliminates the fbcon takeover race entirely: virtio-gpu (or QXL) is the first and only DRM driver, fbcon binds once cleanly, before agetty starts.
- `/proc/cmdline` carries the full `GRUB_CMDLINE_LINUX_DEFAULT` including `fbcon=nodefer` (bonus — but moot now that simpledrm isn't in the picture).

**Tradeoff accepted:** no UEFI Secure Boot. Irrelevant for a cloud-server workflow where the OS trust model is based on the qcow2's sha256 anyway.

**Secondary fix: virtio-gpu instead of QXL.** QXL is a legacy SPICE-specific video device (2010-era Red Hat stack, X11-oriented). virtio-gpu is the modern paravirtualized-GPU default for Linux guests under KVM/QEMU (kernel 4.16+, 2018). Native virtio_gpu DRM driver, Wayland-native, simpler config (no QXL ram_size + vram_size + vram64_size_mb + vgamem_mb quartet — just `vram:` and `heads:`). Still fully SPICE-compatible. See `/ov-dev:libvirt-renderer` for the video-model decision matrix.

**Do NOT attempt the guest-side fix** (regenerate BOOTX64.EFI, add `fbcon=nodefer` via `/etc/default/grub.d/`, sed on `/etc/default/grub`, `grub-install` at first boot, power-cycle reboot). The user explicitly rejected these during plan review: "match libvirt config to the deployed image, not the other way around". The maintainer-authored grub config on vda3 is the source of truth; BIOS boot reads it as-shipped.

## Finding C — SSH flakes during spice-vdagent install: resource sizing, not package pruning

Bisect (live-measured):

| Config | RAM | SSH availability |
|---|---|---|
| no graphics, no spice-vdagent | 2 GiB | ~15 s |
| graphics devices only | 2 GiB | ~15 s |
| graphics + `packages: [spice-vdagent]` + `systemctl enable --now spice-vdagentd` | **2 GiB** | **never up in 180 s** |

Root cause: `pacman -S spice-vdagent` pulls in GTK3 + X11 (~200 MB download, ~1 GB installed). Cloud-init's `packages:` module blocks on that + `pacman-key --populate archlinux` (CPU-heavy keyring build) simultaneously. At 2 GiB RAM + 2 cpus, the guest is starved — memory pressure triggers OOM-adjacent behavior, pacman's fsync cycles compete with sshd for IO, sshd either never finishes listening or accepts TCP but resets during kex.

**Fix: host has 61 GiB RAM / 16 cores / 184 GB free. Give the VM 8 GiB / 4 cpus / 40 GiB.** Comfortably holds pacman's keyring + spice-vdagent GTK3/X11 install + kernel caches without eviction pressure. With this sizing, SSH comes up in ≤ 20 s and spice-vdagent is active within 45 s — same as the minimal-config baseline.

**Do NOT prune spice-vdagent from `packages:`** — it provides clipboard sync + resolution negotiation in the SPICE console, standard VDI features. Resource sizing is the correct fix.

**General heuristic**: check `free -h` / `nproc` / `df -h` before sizing a VM. Allocating <10% of host RAM and <25% of host cores is "frugal". Starving a VM to save host capacity the host isn't using is a bug, not thrift.

## Finding D — SSH key idempotency across rebuilds

`ov vm build` + `ov vm create` called repeatedly should NOT regenerate the SSH keypair. Implementation idempotency lives in `ov/vm_cloud_image.go::generateSSHKeypair` — it checks for `~/.local/share/ov/vm/ov-<name>/id_ed25519.pub` before creating. See `/ov-dev:vm-deploy-target` for the state persistence flow (VmDeployState in `deploy.yml`).

## Adopt pattern in action

`source.base_user: arch` triggers the cloud-init renderer to emit:

```yaml
# Rendered user-data (via /ov-dev:cloud-init-renderer composeUsers)
users:
  - default                               # cloud-init keeps distro-default account untouched
  - name: arch
    ssh_authorized_keys:
      - ssh-ed25519 AAAA…                 # generated pubkey from ~/.local/share/ov/vm/ov-arch/id_ed25519.pub
```

cloud-init appends the pubkey to `/home/arch/.ssh/authorized_keys` without calling `useradd`, rewriting sudoers, or changing the shell. `spec.ssh.user` defaults to `arch`, so `ov vm ssh arch` connects as `arch@127.0.0.1:2224`. Full parity with the container-side `base_user:` + `user_policy: adopt` pattern (`/ov-build:image` "user_policy").

## Verification recipes

### Rebuild + boot
```bash
virsh -c qemu:///session destroy ov-arch 2>/dev/null
virsh -c qemu:///session undefine --nvram ov-arch 2>/dev/null
rm -f ~/.local/share/ov/vm/ov-arch/nvram.fd
rm -f output/qcow2/{disk.qcow2,seed.iso}
ov vm build arch
ov vm create arch --no-auto-detect
```

### BIOS firmware + virtio-gpu took effect
```bash
virsh -c qemu:///session dumpxml ov-arch \
  | grep -E "<loader|<nvram|<firmware|<type arch|<model type"
```
Pass: NO `<loader>` / `<nvram>` / `<firmware>` elements; `<model type='virtio'>` inside `<video>`.

### No simpledrm race
```bash
ssh -i ~/.local/share/ov/vm/ov-arch/id_ed25519 -p 2224 arch@127.0.0.1 \
  'sudo dmesg | grep -iE "simpledrm|virtio_gpu|drm"' | head -10
```
Pass: `virtio_gpu` as primary, NO `simpledrm` lines, NO `fbcon: Deferring console take-over`.

### SSH up in ≤ 60 s
```bash
START=$(date +%s)
for i in $(seq 1 20); do
  ssh -i ~/.local/share/ov/vm/ov-arch/id_ed25519 -p 2224 -o ConnectTimeout=3 -o BatchMode=yes \
      arch@127.0.0.1 'echo UP' 2>&1 | grep -q UP && break
  sleep 5
done
echo "SSH-UP-AT: $(( $(date +%s) - START )) s"
```
Pass: ≤ 60 s (previous starved-config was "never up in 180 s").

### Disk/FS at full 40 GiB
```bash
qemu-img info -U output/qcow2/disk.qcow2 | grep "virtual size"
ssh ... arch@127.0.0.1 '
  sudo fdisk -l /dev/vda | head -1
  lsblk -n -o NAME,SIZE /dev/vda3
  sudo btrfs filesystem show | grep size
  df -h /
'
```
All four layers report ~40 GiB.

### spice-vdagent active
```bash
ssh ... arch@127.0.0.1 'systemctl is-active spice-vdagentd; pacman -Q spice-vdagent'
```
Pass: `active` + version printed.

## Cross-References

- `/ov-vms:vms` — VmSpec authoring reference (schema, source.kind, adopt pattern)
- `/ov-advanced:vm` — VM lifecycle commands + BIOS/UEFI decision matrix + video model choice
- `/ov-build:migrate` — `ov migrate vm-spec` legacy conversion
- `/ov-core:deploy` — `ov deploy add vm:arch <layer>` for in-guest layer application
- `/ov-dev:vm-spec` — Go types and validation rules
- `/ov-dev:libvirt-renderer` — `<backend type='passt'/>` for portForward, virtio-gpu video model
- `/ov-dev:cloud-init-renderer` — `composeUsers` adopt-merge, seed ISO, `ov_install.strategy: auto`
- `/ov-dev:ovmf` — why `firmware: bios` skips `<loader>`/`<nvram>` emission entirely
- `/ov-dev:vm-deploy-target` — SSH key idempotency, VmDeployState persistence
- `/ov-foundation:cloud-init` — guest-side cloud-init layer (complementary to host-side emission)
