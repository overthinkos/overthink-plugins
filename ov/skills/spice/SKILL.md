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

- `--address host:port` — bypass vms.yml lookup; targets an arbitrary SPICE server.
- `--password SECRET` — SPICE ticket for `--address` mode.

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
# Start a VM with SPICE graphics (arch-cloud-base ships this by default).
ov vm start arch-cloud-base

# Handshake + report channels.
ov test spice status arch-cloud-base
# → connected: 127.0.0.1:5901
#   display:   1280x800
#   inputs:    ready

# Native SPICE screenshot (not libvirt DomainScreenshot).
ov test spice screenshot arch-cloud-base /tmp/out.png
# → Screenshot saved to /tmp/out.png (1280x800, native SPICE display decode)

# Drive the login.
ov test spice key arch-cloud-base return
ov test spice type arch-cloud-base arch
ov test spice key arch-cloud-base tab
ov test spice type arch-cloud-base "my-password"
ov test spice key arch-cloud-base return
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
