---
name: ov:libvirt
description: libvirt-RPC test commands — `ov test libvirt <vm> …` for VM info, framebuffer screenshots, send-key, passwd, QMP, qemu-guest-agent client, snapshots, events.
allowed-tools: Bash, Read
---

MUST be invoked before any work involving: `ov test libvirt` commands,
libvirt RPC-based VM inspection/management, qemu-guest-agent probing,
VM snapshots, domain event watching, or any task that needs
VM-introspection surfaces beyond what `ov vm` (lifecycle) provides.

## Command surface

**Top-level:**
```
ov test libvirt list                                # all domains on the session
ov test libvirt info       <vm>                     # state + graphics + uptime + agent
ov test libvirt screenshot <vm> [FILE] [--screen N] # libvirt DomainScreenshot → PNG
ov test libvirt send-key   <vm> KEYS…               # keyboard injection (Linux keycode set)
ov test libvirt passwd     <vm> [PASSWORD]          # live graphics password patch
ov test libvirt qmp        <vm> COMMAND [JSON_ARGS] # QMP passthrough
ov test libvirt domain-xml <vm>                     # dump live XML
ov test libvirt console    <vm>                     # (stub) serial console — use `virsh console`
ov test libvirt events     [<vm>]                   # poll lifecycle state transitions
```

**`guest` subgroup — qemu-guest-agent client:**
```
ov test libvirt guest ping       <vm>
ov test libvirt guest info       <vm>          # agent capability list
ov test libvirt guest os-info    <vm>          # distro/kernel
ov test libvirt guest time       <vm>          # guest clock + host delta
ov test libvirt guest hostname   <vm>
ov test libvirt guest users      <vm>
ov test libvirt guest interfaces <vm>          # IPs + MACs + stats
ov test libvirt guest disks      <vm>          # block devices
ov test libvirt guest fsinfo     <vm>          # mounted filesystems
ov test libvirt guest vcpus      <vm>          # guest-side online state
ov test libvirt guest exec       <vm> -- CMD ARGS…
ov test libvirt guest file read  <vm> PATH     # cat a guest file
ov test libvirt guest file write <vm> PATH     # read stdin, write to guest
ov test libvirt guest fsfreeze status|freeze|thaw <vm>
ov test libvirt guest fstrim     <vm>
```

**`snapshot` subgroup:**
```
ov test libvirt snapshot list   <vm>
ov test libvirt snapshot create <vm> NAME [--desc] [--disk-only]
ov test libvirt snapshot info   <vm> NAME
ov test libvirt snapshot revert <vm> NAME
ov test libvirt snapshot delete <vm> NAME
```

## What it does

Every verb maps to one go-libvirt RPC (plus, for `guest *`, the
QEMUDomainAgentCommand passthrough). The command name mirrors the
libvirt/virsh vocabulary so skills translate cleanly. Design
choices:

- **Target resolution** identical to `ov test spice`: vms.yml entity
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
ov test libvirt list
ov test libvirt info arch-cloud-base

# Framebuffer capture (independent of SPICE wire state).
ov test libvirt screenshot arch-cloud-base /tmp/fb.png

# Keyboard injection via libvirt (alternate path to `ov test spice key`).
ov test libvirt send-key arch-cloud-base ctrl+alt+F2

# Live graphics password update.
ov test libvirt passwd arch-cloud-base hunter2

# QMP escape hatch.
ov test libvirt qmp arch-cloud-base query-status

# qemu-guest-agent (only if the guest has qemu-guest-agent installed).
ov test libvirt guest ping arch-cloud-base
ov test libvirt guest exec arch-cloud-base -- uname -a

# Transactional testing: freeze → snapshot → thaw.
ov test libvirt guest fsfreeze freeze arch-cloud-base
ov test libvirt snapshot create arch-cloud-base pre-exp
ov test libvirt guest fsfreeze thaw   arch-cloud-base
# … exercise something destructive …
ov test libvirt snapshot revert arch-cloud-base pre-exp
ov test libvirt snapshot delete arch-cloud-base pre-exp
```

## Agent prerequisite

`guest *` subcommands require `qemu-guest-agent` installed and
running in the guest AND a `<channel type='unix' ... name='org.qemu.guest_agent.0'/>`
in the domain XML. The overthink layer `qemu-guest-agent` drops
this in for bootc VMs; cloud-image VMs need to add it to
`cloud_init.packages:` explicitly.

Use `ov test libvirt info <vm>` — if `Agent: false`, the agent is
not reachable.

## Architecture split (vs. `ov test spice`)

- **`ov test libvirt`** — libvirt RPC only. Every byte goes to
  libvirtd. Works regardless of the VM's graphics protocol.
- **`ov test spice`** — SPICE wire only. Proves the SPICE protocol
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
