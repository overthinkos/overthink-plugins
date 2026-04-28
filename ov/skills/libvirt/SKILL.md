---
name: ov:libvirt
description: libvirt-RPC test commands — `ov eval libvirt <vm> …` for VM info, framebuffer screenshots, send-key, passwd, QMP, qemu-guest-agent client, snapshots, events.
allowed-tools: Bash, Read
---

MUST be invoked before any work involving: `ov eval libvirt` commands,
libvirt RPC-based VM inspection/management, qemu-guest-agent probing,
VM snapshots, domain event watching, or any task that needs
VM-introspection surfaces beyond what `ov vm` (lifecycle) provides.

## Command surface

**Top-level:**
```
ov eval libvirt list                                # all domains on the session
ov eval libvirt info       <vm>                     # state + graphics + uptime + agent
ov eval libvirt screenshot <vm> [FILE] [--screen N] # libvirt DomainScreenshot → PNG
ov eval libvirt send-key   <vm> KEYS…               # keyboard injection (Linux keycode set)
ov eval libvirt passwd     <vm> [PASSWORD]          # live graphics password patch
ov eval libvirt qmp        <vm> COMMAND [JSON_ARGS] # QMP passthrough
ov eval libvirt domain-xml <vm>                     # dump live XML
ov eval libvirt console    <vm>                     # (stub) serial console — use `virsh console`
ov eval libvirt events     [<vm>]                   # poll lifecycle state transitions
```

**Remote libvirt via `--uri`.** Every verb accepts `--uri
qemu+ssh://[user@]host/session` (also honored as `OV_LIBVIRT_URI`). When
set, `ov` opens an SSH connection to the remote host, discovers the
remote virtqemud session socket via `id -u`, and forwards it over the
SSH channel — so `DomainScreenshot`, `DomainSendKey`, QMP, etc. all
work against the remote hypervisor. Example:

```bash
ov eval libvirt info arch --uri qemu+ssh://o.atrawog.org/session
ov eval libvirt screenshot arch --uri qemu+ssh://o.atrawog.org/session - > /tmp/shot.png
```

`<file>` args accept `-` for stdout. Alternatively, run `ov` on the
remote machine with the top-level `--host` flag: `ov --host o test
libvirt info arch`.

**`guest` subgroup — qemu-guest-agent client:**
```
ov eval libvirt guest ping       <vm>
ov eval libvirt guest info       <vm>          # agent capability list
ov eval libvirt guest os-info    <vm>          # distro/kernel
ov eval libvirt guest time       <vm>          # guest clock + host delta
ov eval libvirt guest hostname   <vm>
ov eval libvirt guest users      <vm>
ov eval libvirt guest interfaces <vm>          # IPs + MACs + stats
ov eval libvirt guest disks      <vm>          # block devices
ov eval libvirt guest fsinfo     <vm>          # mounted filesystems
ov eval libvirt guest vcpus      <vm>          # guest-side online state
ov eval libvirt guest exec       <vm> -- CMD ARGS…
ov eval libvirt guest file read  <vm> PATH     # cat a guest file
ov eval libvirt guest file write <vm> PATH     # read stdin, write to guest
ov eval libvirt guest fsfreeze status|freeze|thaw <vm>
ov eval libvirt guest fstrim     <vm>
```

**`snapshot` subgroup:**
```
ov eval libvirt snapshot list   <vm>
ov eval libvirt snapshot create <vm> NAME [--desc] [--disk-only]
ov eval libvirt snapshot info   <vm> NAME
ov eval libvirt snapshot revert <vm> NAME
ov eval libvirt snapshot delete <vm> NAME
```

## What it does

Every verb maps to one go-libvirt RPC (plus, for `guest *`, the
QEMUDomainAgentCommand passthrough). The command name mirrors the
libvirt/virsh vocabulary so skills translate cleanly. Design
choices:

- **Target resolution** identical to `ov eval spice`: vms.yml entity
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
ov eval libvirt list
ov eval libvirt info arch

# Framebuffer capture (independent of SPICE wire state).
ov eval libvirt screenshot arch /tmp/fb.png

# Keyboard injection via libvirt (alternate path to `ov eval spice key`).
ov eval libvirt send-key arch ctrl+alt+F2

# Live graphics password update.
ov eval libvirt passwd arch hunter2

# QMP escape hatch.
ov eval libvirt qmp arch query-status

# qemu-guest-agent (only if the guest has qemu-guest-agent installed).
ov eval libvirt guest ping arch
ov eval libvirt guest exec arch -- uname -a

# Transactional testing: freeze → snapshot → thaw.
ov eval libvirt guest fsfreeze freeze arch
ov eval libvirt snapshot create arch pre-exp
ov eval libvirt guest fsfreeze thaw   arch
# … exercise something destructive …
ov eval libvirt snapshot revert arch pre-exp
ov eval libvirt snapshot delete arch pre-exp
```

## Agent prerequisite

`guest *` subcommands require `qemu-guest-agent` installed and
running in the guest AND a `<channel type='unix' ... name='org.qemu.guest_agent.0'/>`
in the domain XML. The overthink layer `qemu-guest-agent` drops
this in for bootc VMs; cloud-image VMs need to add it to
`cloud_init.packages:` explicitly.

Use `ov eval libvirt info <vm>` — if `Agent: false`, the agent is
not reachable.

## Architecture split (vs. `ov eval spice`)

- **`ov eval libvirt`** — libvirt RPC only. Every byte goes to
  libvirtd. Works regardless of the VM's graphics protocol.
- **`ov eval spice`** — SPICE wire only. Proves the SPICE protocol
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
  hard dep of the `ov-git` PKGBUILD.
