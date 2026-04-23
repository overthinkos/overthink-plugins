---
name: ov:spice
description: SPICE wire-level client for VMs — `ov test spice <vm>` handshake, inputs, native display screenshots via the Shells-com/spice library.
allowed-tools: Bash, Read
---

MUST be invoked before any work involving: `ov test spice` commands,
SPICE protocol debugging, libvirt-VM display testing via the SPICE
wire, or anything that needs to *prove* a SPICE server is speaking
the protocol correctly (not just that the TCP port is open).

## Command surface

```
ov test spice status     <vm>              # handshake + channel enumeration
ov test spice screenshot <vm> [FILE]       # native SPICE display-channel decode → PNG
ov test spice click      <vm> X Y          # mouse press/release via inputs channel
ov test spice mouse      <vm> X Y          # pointer move (no click)
ov test spice type       <vm> TEXT         # type text as PC-AT scancodes
ov test spice key        <vm> NAME         # press one named key (Return, Escape, F2, …)
ov test spice cursor     <vm> [FILE]       # capture cursor bitmap + position
```

Global flags on every subcommand:

- `--address host:port` — bypass vms.yml lookup; targets an arbitrary TCP SPICE server.
- `--socket /path` — bypass vms.yml lookup; targets an arbitrary UNIX-socket SPICE server.
- `--password SECRET` — SPICE ticket for `--address` mode.
- `--uri qemu+ssh://[user@]host/session` — resolve the VM on a remote libvirt host. `ov` auto-opens an SSH tunnel and forwards the remote SPICE socket (or TCP port) to a local endpoint for the lifetime of the command. Also accepts the `OV_LIBVIRT_URI` env var.

Screenshot/cursor `<file>` args accept `-` to write PNG bytes to stdout:
`ov test spice screenshot arch - > /tmp/shot.png`. Status messages
go to stderr so stdout stays binary-clean — pipeline-friendly.

## Remote libvirt (qemu+ssh://)

When `--uri qemu+ssh://…` is set, `ov test spice` runs locally but pokes a
remote libvirt. Libvirt RPC is tunneled over the SSH control channel for
free; the SPICE display channel needs a side tunnel, which `ov` opens
transparently:

```bash
ov test spice status arch --uri qemu+ssh://o.atrawog.org/session
# → opens SSH → discovers remote virtqemud socket via `id -u` → forwards
#   the SPICE UNIX socket → dials it → prints channel enumeration.
```

For VMs that declare `<listen type='socket'/>` (the default for
arch after the socket-listen cutover), `ov` forwards the UNIX
socket. For TCP-listener VMs, `ov` opens a 127.0.0.1:<random> forward.

Alternative: `ov --host o test spice status arch` runs `ov` on
the remote machine itself — no side tunnel needed because the VM's SPICE
socket is local to the remote host. Useful when you want artifacts to stay
remote, or when the SPICE server doesn't bind loopback on the remote side.

GUI clients (virt-manager, `remote-viewer --connect qemu+ssh://…`) don't
need any ov involvement for socket listeners — they auto-forward via
libvirt RPC fd-passing. See `/ov-vms:arch` "Connecting from a
remote workstation" for the complete story.

## What it does (and doesn't)

- **Speaks SPICE on the wire.** Main channel handshake, auth (None or
  SPICE_TICKET), channel enumeration (Display/Inputs/Cursor/Playback/
  Record/Webdav), display channel with native QUIC/GLZ/LZ/LZ4 image
  decode, input channel for key/mouse events.
- **Pure-Go + cgo audio deps.** Uses `github.com/Shells-com/spice`.
  Audio channels drag in `portaudio` + `opusfile` (Arch packages
  listed in `pkg/arch/PKGBUILD`); these are required at build + run
  time but the CLI itself never plays/records audio.
- **Target resolution.** Default path: `ov test spice <vm>` loads
  vms.yml, finds the running libvirt domain, parses live XML via
  `libvirtxml.Domain`, and extracts the SPICE host/port/passwd from
  the `<graphics type="spice">` element (honoring autoport='yes').
- **Not implemented.** Audio playback/record (no user story).
  Complex agent-channel operations (clipboard, resolution change);
  the upstream library exposes them but the CLI doesn't wrap them
  yet — use the libvirt subcommands (`libvirt passwd`, `libvirt
  guest exec`) for the corresponding management ops.

## End-to-end example

```bash
# Start a VM with SPICE graphics (arch ships this by default).
ov vm start arch

# Handshake + report channels.
ov test spice status arch
# → connected: 127.0.0.1:5901
#   display:   1280x800
#   inputs:    ready

# Native SPICE screenshot (not libvirt DomainScreenshot).
ov test spice screenshot arch /tmp/out.png
# → Screenshot saved to /tmp/out.png (1280x800, native SPICE display decode)

# Drive the login.
ov test spice key arch return
ov test spice type arch arch
ov test spice key arch tab
ov test spice type arch "my-password"
ov test spice key arch return
```

## Architecture split (vs. `ov test libvirt`)

The two commands are single-protocol by design:

- **`ov test spice`** — every byte flows through the SPICE wire.
  Use when the thing under test is "is SPICE itself healthy?".
- **`ov test libvirt`** — every call goes through libvirtd RPC.
  Use when the thing under test is "is the VM working?" (framebuffer
  capture, keyboard injection, snapshots, domain state).

For input testing, prefer `ov test spice type/key/click` to prove
the SPICE wire delivers input to the guest. For display testing,
compare `ov test spice screenshot` against `ov test libvirt
screenshot` — if both render the same pixels, the SPICE server +
the guest framebuffer agree.

## Implementation pointers

- `ov/spice.go` — Kong command tree, per-verb structs.
- `ov/spice_session.go` — thin wrapper over Shells-com/spice's
  `Connector`/`Driver` interfaces. The Driver stub captures
  display/cursor updates into a sync.Mutex-guarded field; the CLI
  reads them back for screenshot/cursor verbs.
- `ov/vm_target.go` — shared target resolution (vms.yml → libvirt
  domain → live XML → SPICE address).
- PC-AT scancode table is inline in `ov/spice.go` (friendly keyname
  → scancode for `type` and `key`).

## Dependencies

- `github.com/Shells-com/spice` (MIT) — SPICE client library.
- `portaudio` + `opusfile` Arch system packages (cgo dependencies
  pulled in by the library's audio channels).
- `libvirt.org/go/libvirtxml` — XML parsing for SPICE address
  extraction.
- `github.com/digitalocean/go-libvirt` — domain lookup.
