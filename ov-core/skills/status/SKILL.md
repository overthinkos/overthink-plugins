---
name: status
description: |
  Service status display with tool probes and device detection.
  MUST be invoked before any work involving: ov status command, checking container state, tool availability, port mapping, or JSON status output.
---

# ov status -- Service Status Display

## Overview

Display running container status in table or detail mode. Built around a
**Collector** that does one batched `podman ps` + one batched `podman
inspect` for every ov-* container, then fans containers out across a
`runtime.NumCPU()*2` worker pool. Each container's host probes (CDP HTTP,
VNC TCP) run in parallel goroutines; all in-container ("guest") probes
(supervisord, dbus, ov, wl, sway) batch into a single `podman exec sh -c`
invocation. End-to-end wall-clock for ~20 containers is ~1–3 s (was ~55 s
before the 2026-05-02 redesign).

Source layout:

- `ov/status.go` — thin `StatusCmd` + dispatch.
- `ov/status_engine.go` — `EngineClient` (only place that touches
  podman/docker), `ContainerSnapshot`, structured `PortMapping`.
- `ov/status_collector.go` — `Collector.All` / `Collector.Single` +
  worker pool; `parseQuadletDescription`; deploy.yml lookup.
- `ov/status_probes.go` — `Probe` / `HostProbe` / `GuestProbe` interfaces
  and the 7 concrete probes (`SupervisordProbe`, `DbusProbe`, `OvProbe`,
  `WlProbe`, `SwayProbe` are guest; `CdpProbe`, `VncProbe` are host).
  `runGuestProbes` builds a single concatenated shell script with
  per-probe markers and splits the stdout chunks back out.
- `ov/status_render.go` — `RenderTable` / `RenderDetail` / `RenderJSON`
  + cell formatters; `formatTunnelSummary`.
- `ov/status_reap.go` — `ReapOrphansCmd` (was `ov status --reap-orphans`,
  now its own top-level `ov reap-orphans` command).

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Table (running) | `ov status` | Show all running services |
| Table (all) | `ov status --all` | Include stopped and enabled services |
| Detail | `ov status <image>` | Key-value detail for one service |
| Detail (instance) | `ov status <image> -i <inst>` | Key-value detail for one instance |
| JSON output | `ov status --json` | Machine-readable JSON (structured ports) |
| Reap orphans | `ov reap-orphans` | Clean up ephemerals whose underlying resource is gone |

## Table Mode Columns

| Column | Description |
|--------|-------------|
| IMAGE | `image` for base deploys, `image/instance` for multi-instance (matches `deployKey` shape) |
| STATUS | `running` / `stopped` / `enabled` / `failed` / `dead` / `paused` |
| PORTS | Sorted, deduped host port numbers from runtime `podman ps` (deploy.yml / image labels are fallbacks for non-running rows) |
| TUNNEL | `provider (all ports)` / `provider (ports H,H,H)` / `-` — read from deploy.yml |
| DEVICES | Compact tokens (`gpu`, `dri`, `kvm`, `fuse`, `tun`) sorted alphabetically |
| TOOLS | Live-probed tools — port-based show `name:port`, socket-based show just `name` |

## Detail Mode Fields

`ov status <image>` shows:

| Field | Example |
|-------|---------|
| Image | `ghcr.io/overthinkos/jupyter:latest` |
| Status | `running` |
| Container | `ov-jupyter` |
| Mode | `quadlet` |
| Ports | `8888/tcp -> 127.0.0.1:8888` |
| Devices | `nvidia (CUDA)` |
| Tools | `cdp:9222, vnc:5900, sway, wl` |
| Volumes | `data: bind /home/user/data` |
| Network | `host` |
| Tunnel | `cloudflare: jupyter.example.com` |

## Tool Probes

Two probe kinds. Host probes (cdp, vnc) run from the operator host using
the snapshot's `HostPortFor(ctrPort, proto)` lookup — no extra `podman
port` / `podman inspect` calls. Guest probes (supervisord, dbus, ov, wl,
sway) batch into ONE `podman exec sh -c` per container; each probe's
snippet emits a `KEY=value` line that its `Parse` recognises. The batcher
delimits sections with `===PROBE:<name>===` / `===PROBE_END:<name>===`
markers.

| Tool | Kind | Snippet / Probe | Display |
|------|------|-----------------|---------|
| supervisord | guest | `command -v supervisorctl && supervisorctl status` | `supervisord` (with `N/M running` detail in detail view) |
| dbus | guest | `pgrep -x dbus-daemon` + scan for `swaync`/`mako`/`dunst` | `dbus` (notifier list in detail view) |
| ov | guest | `command -v ov && ov version` | `ov` (CalVer detail) |
| wl | guest | `command -v wtype/wlrctl/grim/pixelflux-screenshot` | `wl` (detail lists available tools) |
| sway | guest | discover SWAYSOCK then `swaymsg -t get_outputs` | `sway` (output dimensions in detail) |
| cdp | host | HTTP GET `:HOST_PORT/json` (port from snapshot) | `cdp:HOST_PORT` |
| vnc | host | TCP dial + RFB banner read | `vnc:HOST_PORT` |

Adding a new probe: implement `HostProbe` (network) or `GuestProbe`
(in-container) in `status_probes.go` and register in the package-level
`hostProbes` / `guestProbes` slice. No other file needs editing.

## JSON output schema

`ov status --json` emits an array of `ContainerStatus` objects. The
2026-05-02 redesign changed `ports` from `[]string` to a structured
array — programmatic consumers must read the new shape (no compat shim):

```json
{
  "image": "selkies-desktop",
  "instance": "work",
  "status": "running",
  "container": "ov-selkies-desktop-work",
  "ports": [
    { "host_ip": "127.0.0.1", "host_port": 9240, "container_port": 9222, "protocol": "tcp" }
  ],
  "tunnel": "tailscale (all ports)",
  "tools": [ { "name": "cdp", "status": "ok", "port": 9240, "detail": "3 tabs" } ],
  "run_mode": "quadlet"
}
```

Single-image (`ov status <image> -i <inst> --json`) emits one object,
not an array.

## Source-of-truth priority for the PORTS column

1. Runtime `podman ps` mappings (`ContainerSnapshot.Ports`) — wins for
   running containers.
2. `deploy.yml` `ports:` (parsed via canonical `ParsePortMapping` —
   handles the `127.0.0.1:H:C/proto` IPv4-prefixed form correctly) —
   used when runtime data is empty.
3. Image-label fallback (`ResolveNewestLocalCalVer` + `ExtractMetadata`)
   — last resort for stopped/enabled rows. **Lookup uses the BASE image
   name from the parsed quadlet description** (e.g. `selkies-desktop`),
   not the joined container name (`selkies-desktop-185.52.136.164`),
   which was the 2026-05-02 bug.

## Known display issue: `Volumes:` reads from image labels

The `Volumes:` field is populated from the image's OCI labels
(`ExtractMetadata` in `ov/status_collector.go`), not from the live
container's actual mounts. This means a volume deployed with `--bind
<name>=<path>` will appear in `ov status` as `ov-<image>-<volume> ->
/container/path` (the image default from the OCI label), not as
`<host-path> -> /container/path` (the actual runtime bind mount).

The running container is functionally correct — only the display is misleading. Authoritative sources for the live volume backing:

```bash
# What the live container is actually mounting
podman inspect <container> --format '{{range .Mounts}}{{.Type}}:{{.Source}}->{{.Destination}} {{"\n"}}{{end}}'

# The deploy.yml record (what ov config resolved)
ov deploy show <image>
```

Use either of the above when you need to confirm that a `--bind` / `--encrypt` override actually took effect.

## Usage

```bash
# Quick overview of all running services
ov status

# Include stopped services
ov status --all

# Detailed info for one service
ov status jupyter

# JSON for scripting
ov status --json | jq '.[] | select(.status == "running")'
```

## Cross-References

- `/ov-build:pull` -- Prerequisite: fetch the image into local storage; handles remote refs (`@github.com/...`) and the `ErrImageNotLocal` recovery path

- `/ov-core:start` -- start a service
- `/ov-core:stop` -- stop a service (via `/ov-core:service`)
- `/ov-core:logs` -- view service logs (via `/ov-core:service`)
- `/ov-core:service` -- full service lifecycle management

## Live-deploy verification is mandatory (see `/ov-build:eval` 10 standards)

Changes that touch this verb's output must reach a healthy deployment on a target explicitly marked `disposable: true` (see `/ov-dev:disposable`). Use `ov rebuild <name>` to destroy + rebuild unattended on any disposable target. Never experiment on a non-disposable deploy — set up a disposable one first with `ov deploy add <name> <ref> --disposable` or mark a VM in vms.yml.

**After committing the source-level fix, `ov rebuild` the disposable target ONCE MORE from clean and re-run the full verification.** A fix that passes only on a hand-patched target is not a real fix — it's a regression waiting for the next unrelated rebuild. Paste BOTH the exploratory-pass output and the fresh-rebuild-pass output into the conversation.

Unit tests + a clean compile are necessary but not sufficient. See CLAUDE.md R1–R10.
