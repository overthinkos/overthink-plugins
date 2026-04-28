---
name: ov:ssh
description: Generic SSH support for ov — `--host <alias>` re-execs any command on a remote machine; `ov ssh tunnel` exposes remote SPICE/VNC endpoints on the local host for external GUI apps.
allowed-tools: Bash, Read
---

MUST be invoked before any work involving: `ov --host <alias|target>`
remote execution, `ov ssh tunnel` port forwarding for external SPICE/VNC
viewers, or managing host aliases via `ov settings set hosts.<alias>`.

## Two orthogonal surfaces

### 1. `ov --host <alias>` — re-exec any ov command on a remote machine

Set `--host` (or `OV_HOST`) at the **top level** of any `ov` invocation.
`ov` shells out to the system `ssh` binary, runs `ov <rest of argv>` on
the remote host, and streams stdin/stdout/stderr through. Exit code
propagates.

```bash
# Alias setup (once per workstation):
ov settings set hosts.o o.atrawog.org
ov settings set hosts.prod admin@prod.example.com:2222

# Any ov verb works:
ov --host o status
ov --host o start openclaw
ov --host o vm list
ov --host o deploy add host fedora-coder
ov --host o test spice status arch
ov --host o test spice screenshot arch - > /tmp/local.png   # stdout pipes back
```

**LocalOnly commands are NOT re-execed**, even when `--host` is set:
`ov settings …`, `ov version`, `ov ssh tunnel …`. These manage the local
workstation (settings file, CLI version, local tunnel listener) and
would be meaningless on the remote host.

**Transport:** system `ssh` binary via `os/exec`, so `~/.ssh/config`,
agent forwarding, and ControlMaster all work transparently. If your
target needs a specific key, set it in `~/.ssh/config` — `ov` stays out
of SSH authentication.

**Client-only flags stripped** before re-exec: `--host`, `--dir` / `-C`,
`--repo`, `--kdbx`. These are workstation-local concerns and must not be
forwarded to the remote side.

### 2. `ov ssh tunnel` — expose a remote VM's display for external GUI apps

For apps that aren't `ov` (virt-viewer, remote-viewer with a bare URL,
TigerVNC, Spicy), open an SSH-forwarded local endpoint:

```bash
ov ssh tunnel spice <vm> [--uri qemu+ssh://user@host/session] [--tcp]
ov ssh tunnel vnc   <vm> [--uri qemu+ssh://user@host/session] [--tcp]
```

Default mode preserves the wire format: UNIX socket in, UNIX socket out
(local path under `/tmp/ov-tunnel-<id>.sock`). `--tcp` forces a
127.0.0.1:<random> TCP listener for clients that don't understand
`spice+unix://` / `vnc+unix://`.

```
$ ov ssh tunnel spice arch --uri qemu+ssh://o.atrawog.org/session
spice tunnel: spice+unix:///tmp/ov-tunnel-8e4c.sock
Connect with: remote-viewer spice+unix:///tmp/ov-tunnel-8e4c.sock
Press Ctrl-C to close the tunnel.
```

Blocks until SIGINT/SIGTERM; closes listener + SSH client cleanly on
exit.

**Not needed for virt-manager or `remote-viewer --connect qemu+ssh://`**
— those auto-forward UNIX-socket listeners through libvirt's RPC
fd-passing, with zero ov involvement. See `/ov-vms:arch`.

## Alias management

| Command | Effect |
|---|---|
| `ov settings set hosts.<alias> <ssh-target>` | create/update alias |
| `ov settings get hosts.<alias>` | print resolved target |
| `ov settings reset hosts.<alias>` | delete alias |
| `ov settings list` | show all settings including host_aliases map |

`<ssh-target>` forms: `host`, `user@host`, `user@host:port`. When
resolving, plain words that look like aliases (no `@`, no `.`) are
looked up in `hosts.*`; anything else is treated as a raw ssh target
and passed through.

## When NOT to use `--host`

- When you want artifacts (screenshots, recordings) to land in the local
  filesystem → use `ov eval libvirt|spice|vnc --uri qemu+ssh://…`
  instead; it runs ov locally and forwards the display channel over SSH.
- When `ov` isn't installed on the remote machine → use `--uri` or
  `ov ssh tunnel`.

## Cross-References

- `/ov-vms:arch` — "Connecting from a remote workstation" —
  the canonical worked example across all three paths.
- `/ov:settings` — `hosts.<alias>` key schema.
- `/ov:spice` — `--uri` + `--socket` flags on `ov eval spice`.
- `/ov:libvirt` — `--uri` flag on every `ov eval libvirt` verb.
- `/ov:vnc` — `ov eval vnc vm <name> …` subcommand group.
