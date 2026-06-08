---
name: libvirt
description: libvirt-RPC test commands — `charly eval libvirt <vm> …` for VM info, framebuffer screenshots, send-key, passwd, QMP, qemu-guest-agent client, snapshots, events.
allowed-tools: Bash, Read
---

MUST be invoked before any work involving: `charly eval libvirt` commands,
libvirt RPC-based VM inspection/management, qemu-guest-agent probing,
VM snapshots, domain event watching, or any task that needs
VM-introspection surfaces beyond what `charly vm` (lifecycle) provides.

## Command surface

**Top-level:**
```
charly eval libvirt list                                # all domains on the session
charly eval libvirt info       <vm>                     # state + graphics + uptime + agent
charly eval libvirt screenshot <vm> [FILE] [--screen N] # libvirt DomainScreenshot → PNG
charly eval libvirt send-key   <vm> KEYS…               # keyboard injection (Linux keycode set)
charly eval libvirt passwd     <vm> [PASSWORD]          # live graphics password patch
charly eval libvirt qmp        <vm> COMMAND [JSON_ARGS] # QMP passthrough
charly eval libvirt domain-xml <vm>                     # dump live XML
charly eval libvirt console    <vm>                     # (stub) serial console — use `virsh console`
charly eval libvirt events     [<vm>]                   # poll lifecycle state transitions
```

**Remote libvirt via `--uri`.** Every verb accepts `--uri
qemu+ssh://[user@]host/session` (also honored as `OV_LIBVIRT_URI`). When
set, `ov` opens an SSH connection to the remote host, discovers the
remote virtqemud session socket via `id -u`, and forwards it over the
SSH channel — so `DomainScreenshot`, `DomainSendKey`, QMP, etc. all
work against the remote hypervisor. Example:

```bash
charly eval libvirt info arch --uri qemu+ssh://o.atrawog.org/session
charly eval libvirt screenshot arch --uri qemu+ssh://o.atrawog.org/session - > /tmp/shot.png
```

`<file>` args accept `-` for stdout. Alternatively, run `ov` on the
remote machine with the top-level `--host` flag: `charly --host o test
libvirt info arch`.

**`guest` subgroup — qemu-guest-agent client:**
```
charly eval libvirt guest ping       <vm>
charly eval libvirt guest info       <vm>          # agent capability list
charly eval libvirt guest os-info    <vm>          # distro/kernel
charly eval libvirt guest time       <vm>          # guest clock + host delta
charly eval libvirt guest hostname   <vm>
charly eval libvirt guest users      <vm>
charly eval libvirt guest interfaces <vm>          # IPs + MACs + stats
charly eval libvirt guest disks      <vm>          # block devices
charly eval libvirt guest fsinfo     <vm>          # mounted filesystems
charly eval libvirt guest vcpus      <vm>          # guest-side online state
charly eval libvirt guest exec       <vm> -- CMD ARGS…
charly eval libvirt guest file read  <vm> PATH     # cat a guest file
charly eval libvirt guest file write <vm> PATH     # read stdin, write to guest
charly eval libvirt guest fsfreeze status|freeze|thaw <vm>
charly eval libvirt guest fstrim     <vm>
```

**`snapshot` subgroup:**
```
charly eval libvirt snapshot list   <vm>
charly eval libvirt snapshot create <vm> NAME [--desc] [--disk-only]
charly eval libvirt snapshot info   <vm> NAME
charly eval libvirt snapshot revert <vm> NAME
charly eval libvirt snapshot delete <vm> NAME
```

## What it does

Every verb maps to one go-libvirt RPC (plus, for `guest *`, the
QEMUDomainAgentCommand passthrough). The command name mirrors the
libvirt/virsh vocabulary so skills translate cleanly. Design
choices:

- **Target resolution** identical to `charly eval spice`: vm.yml entity
  name → libvirt domain → live XML via `libvirtxml.Domain`. Errors
  are the same across both commands.
- **Framebuffer screenshot** goes through `DomainScreenshot` —
  QEMU's VNC framebuffer capture, returned as PPM, decoded in-process
  and re-encoded as PNG. Independent of whichever graphics protocol
  (SPICE or VNC) the VM exposes on the wire.
- **send-key** uses Linux keycode set via `DomainSendKey`. Friendly
  keyname map in `ov/libvirt_ops.go` covers letters/digits/modifiers/
  arrows/function keys. Supports chord notation (`"ctrl+alt+F2"`).
- **passwd** patches the `<graphics type="spice|vnc" passwd="…">`
  attribute via `libvirtxml`, then calls
  `DomainUpdateDeviceFlags(LIVE)`. `--persistent` also writes to the
  inactive config.
- **QMP** passes JSON commands through unchanged. Flag 0 = QMP;
  HMP not exposed.
- **guest exec** uses `guest-exec` + polls `guest-exec-status` until
  exit. stdout/stderr are base64-decoded from the agent reply and
  written to the CLI's stdout/stderr.
- **events** is a 1-Hz lifecycle poller (not callback-based) — good
  enough for "did the VM actually shut down?" checks; a real event
  subscription will land when go-libvirt's stream APIs are wrapped.

## End-to-end example

```bash
# Sanity.
charly eval libvirt list
charly eval libvirt info arch

# Framebuffer capture (independent of SPICE wire state).
charly eval libvirt screenshot arch /tmp/fb.png

# Keyboard injection via libvirt (alternate path to `charly eval spice key`).
charly eval libvirt send-key arch ctrl+alt+F2

# Live graphics password update.
charly eval libvirt passwd arch hunter2

# QMP escape hatch.
charly eval libvirt qmp arch query-status

# qemu-guest-agent (only if the guest has qemu-guest-agent installed).
charly eval libvirt guest ping arch
charly eval libvirt guest exec arch -- uname -a

# Transactional testing: freeze → snapshot → thaw.
charly eval libvirt guest fsfreeze freeze arch
charly eval libvirt snapshot create arch pre-exp
charly eval libvirt guest fsfreeze thaw   arch
# … exercise something destructive …
charly eval libvirt snapshot revert arch pre-exp
charly eval libvirt snapshot delete arch pre-exp
```

## Agent prerequisite

`guest *` subcommands require `qemu-guest-agent` installed and
running in the guest AND a `<channel type='unix' ... name='org.qemu.guest_agent.0'/>`
in the domain XML. The opencharly layer `qemu-guest-agent` drops
this in for bootc VMs; cloud-image VMs need to add it to
`cloud_init.packages:` explicitly.

Use `charly eval libvirt info <vm>` — if `Agent: false`, the agent is
not reachable.

### Cloud-image VM packages: also include `portaudio`

When a cloud-image VM uses `ov_install.strategy: auto` (the modern
default), VmDeployTarget scps the **host** `ov` binary into the guest
post-boot. The host binary is built with cgo (the
`gordonklaus/portaudio` binding for SPICE audio — see
`pkg/arch/PKGBUILD` `depends=()`), so the guest needs
`libportaudio.so.2` available before the binary runs. Without it,
`charly version` exits 127 with `error while loading shared libraries:
libportaudio.so.2: cannot open shared object file`.

Minimum cloud-image package list for full eval-live coverage:

```yaml
cloud_init:
  packages:
    - sudo
    - spice-vdagent      # for spice-vdagentd
    - qemu-guest-agent   # for `charly eval libvirt guest *` probes
    - portaudio          # for libportaudio.so.2 (host charly binary dep)
```

## Daemon prerequisite (host side)

Every `charly eval libvirt …` and `charly eval spice …` probe routes through
the libvirt user-session daemon socket. The daemon must be running as
a user-service:

```bash
systemctl --user enable --now virtqemud.service     # libvirt ≥ 8 (modular)
# OR (older monolithic):
systemctl --user enable --now libvirtd.service
```

`charly vm create` best-effort starts these units before resolving the
VM backend (see `/charly-vm:vm` "Prereq"). The `charly eval run <bed>` path
for VM-targeting beds (e.g. `eval-k3s-vm`) invokes the same hook. If the
unit doesn't exist (libvirt not installed), the explicit-backend
gate in `resolveVmBackend()` surfaces a clear error pointing at the
remediation.

## Architecture split (vs. `charly eval spice`)

- **`charly eval libvirt`** — libvirt RPC only. Every byte goes to
  libvirtd. Works regardless of the VM's graphics protocol.
- **`charly eval spice`** — SPICE wire only. Proves the SPICE protocol
  path is alive end-to-end.

For ambiguous tests, run both and diff the results.

## Implementation pointers

- `ov/libvirt_cmd.go` — Kong command tree + all verb implementations.
- `ov/libvirt_ops.go` — shared helpers (screenshot PPM→PNG decode,
  key-name → Linux keycode map).
- `ov/libvirt_guest_agent.go` — typed client over
  QEMUDomainAgentCommand with methods for every QGA command in use.
- `ov/vm_target.go` — shared target resolution; `VmTarget.XML`
  gives you the live `libvirtxml.Domain` for fast field lookups.

## Dependencies

- `github.com/digitalocean/go-libvirt` — pure-Go libvirt RPC client.
- `libvirt.org/go/libvirtxml` — typed domain XML parsing/generation.
- The `libvirt` Arch package (libvirtd, session socket) is already a
  hard dep of the `opencharly-git` PKGBUILD.
