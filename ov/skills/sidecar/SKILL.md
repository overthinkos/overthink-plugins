---
name: sidecar
description: |
  MUST be invoked before any work involving: sidecar containers, pod networking,
  Tailscale exit nodes, ov config --sidecar, or deploy.yml sidecars: field.
---

# Sidecar — Deploy-Time Pod Composition

## Overview

Sidecars are additional containers that run alongside an application container in a shared Podman pod. They share the network namespace (localhost connectivity) while maintaining separate filesystems. Sidecar templates are embedded in the `ov` binary — any image can get any sidecar at deploy time without image rebuilds.

## Quick Reference

| Action | Command |
|--------|---------|
| List available sidecars | `ov config --list-sidecars` |
| Attach sidecar | `ov config <image> --sidecar tailscale` |
| Attach with env | `ov config <image> --sidecar tailscale -e TS_HOSTNAME=my-app` |
| Check sidecar status | `podman exec ov-<image>-tailscale tailscale status` |

## Architecture

Sidecar templates are compiled into the `ov` binary via `go:embed` (`ov/sidecar.yml`). At deploy time, `ov config --sidecar <name>` merges the template with per-machine overrides from `deploy.yml`, then generates quadlet files.

```
ov binary (embedded templates)     deploy.yml (per-machine overrides)
        ↓                                    ↓
   sidecar templates              env overrides + secrets
        ↓ MergeSidecars()                    ↓
              ResolveSidecars()
                    ↓
             pod + sidecar + app quadlets
```

### Generated Files

When sidecars are attached, `ov config` generates **3 files** instead of 1:

| File | Purpose |
|------|---------|
| `ov-<image>.pod` | Pod definition: `Network=ov`, ports (`PodmanArgs=-p`), `--shm-size` |
| `ov-<image>-<sidecar>.container` | Sidecar: image, env, caps, devices, secrets, volumes |
| `ov-<image>.container` | App: `Pod=ov-<image>.pod`, no ports/network (pod owns them) |

### Dual Networking

The pod stays on the "ov" bridge network (`Network=ov` in `.pod` file) for container-to-container connectivity. The Tailscale sidecar creates a `tailscale0` tun interface for exit node routing. `--exit-node-allow-lan-access` adds a routing exception (`throw 10.89.0.0/24`) that keeps bridge traffic on the bridge.

```
ov bridge (container-to-container)
┌─────────────────────────────────────┐
│  ov-<image> pod (Network=ov)        │
│  ┌───────────┐  ┌────────────────┐  │
│  │ tailscale │  │ app container  │  │
│  │ sidecar   │  │ :3000, :9222   │  │
│  │ tailscale0│  │                │  │
│  │ (tun)     │  │                │  │
│  └─────┬─────┘  └────────────────┘  │
└────────┼────────────────────────────┘
         │ outbound internet only
    exit node
```

### Env Var Routing

CLI `-e KEY=VALUE` flags are automatically routed: env vars matching a sidecar template's keys (all `TS_*` for tailscale) go to the sidecar's deploy.yml env override. Other vars go to the app container.

### deploy.yml Persistence

`--sidecar` assignments and `-e` overrides are saved to `deploy.yml`. Subsequent `ov config` calls re-read them:

```yaml
images:
  selkies-desktop:
    sidecars:
      tailscale:
        env:
          TS_HOSTNAME: selkies-desktop
          TS_EXTRA_ARGS: "--exit-node=100.80.254.4 --exit-node-allow-lan-access"
```

## Tailscale Sidecar

The built-in tailscale sidecar template (validated against `tailscale/tailscale` containerboot source):

### Environment Variables

| Env var | Default | Purpose |
|---------|---------|---------|
| `TS_STATE_DIR` | `/var/lib/tailscale` | Persistent state (auth, node identity) |
| `TS_AUTH_ONCE` | `true` | Skip re-auth when state exists |
| `TS_USERSPACE` | `false` | **Kernel mode** — required for exit node routing |
| `TS_DEBUG_FIREWALL_MODE` | `nftables` | Force nftables (iptables-legacy fails in rootless podman) |
| `TS_ACCEPT_DNS` | `true` | MagicDNS — split DNS (.ts.net via Tailscale, rest via upstream) |
| `TS_ENABLE_HEALTH_CHECK` | `true` | `/healthz` endpoint |
| `TS_LOCAL_ADDR_PORT` | `[::]:9002` | Health check listen address |
| `TS_AUTHKEY` | Via secret | Auth key (from credential store) |
| `TS_HOSTNAME` | Per-image | Tailscale device name (set via `-e`) |
| `TS_EXTRA_ARGS` | Per-image | Extra `tailscale up` flags (e.g., `--exit-node=<ip>`) |

### Security

- **Capabilities:** `NET_ADMIN` (iptables/nftables, IP forwarding), `SYS_MODULE` (tun/tap kernel module)
- **Devices:** `/dev/net/tun` (TUN/TAP virtual network device)

### State & Secrets

- **Volume:** `ov-<image>-tailscale-state` at `/var/lib/tailscale` — node identity persists across restarts
- **Secret:** `ov-<image>-tailscale-ts-authkey` — provisioned as podman secret from `TS_AUTHKEY` env var (loaded from `.secrets` via `ov secrets gpg env`)

### Exit Node Routing

```bash
# Store auth key for the sidecar's tailnet
ov secrets gpg set TS_AUTHKEY tskey-auth-xxxxxxxxxxxx

# Deploy with exit node
ov config selkies-desktop --sidecar tailscale \
  -e TS_HOSTNAME=selkies-desktop \
  -e "TS_EXTRA_ARGS=--exit-node=100.80.254.4 --exit-node-allow-lan-access"
ov start selkies-desktop

# First time only: exit node must be set via tailscale set
# (TS_EXTRA_ARGS only applies on first auth, not restarts)
podman exec ov-selkies-desktop-tailscale \
  tailscale set --exit-node=100.80.254.4 --exit-node-allow-lan-access

# Verify: pod shows exit node's IP, not host's
podman exec ov-selkies-desktop curl -s ifconfig.me
```

**Prerequisites:**
- Exit node device must advertise as exit node (`tailscale set --advertise-exit-node` on the device)
- Exit node must be **approved** on the sidecar's tailnet admin console
- Exit node IP is the **Tailscale IP** (e.g., `100.80.254.4`), not a public IP

**Persistence:** `tailscale set --exit-node` persists in the state volume. Survives pod restarts without re-configuration.

### Different Tailnet from Host

The sidecar runs its own `tailscaled` daemon in the pod's network namespace with its own state volume. It is completely isolated from the host's Tailscale:

| Aspect | Host Tailscale | Sidecar Tailscale |
|--------|---------------|-------------------|
| State | `/var/lib/tailscale` | Named volume |
| Auth key | Host's | Sidecar's `TS_AUTHKEY` |
| Network NS | Host | Pod (shared with app) |
| Control server | Default | Configurable via `TS_EXTRA_ARGS=--login-server=...` |

### Host Tailnet Port Exposure

The host's `tunnel: tailscale` (configured in images.yml) is independent of the sidecar:
- `ExecStartPost=tailscale serve` runs on the **host**, exposing pod ports on the **host's tailnet**
- The sidecar handles routing on the **sidecar's tailnet**
- Both work simultaneously (dual networking)

### Pod ShmSize

Chrome requires large `/dev/shm`. In pod mode, per-container `ShmSize=` is ignored (pod infra container owns `/dev/shm`). The pod quadlet propagates ShmSize via `PodmanArgs=--shm-size=1g`.

## Cross-References

- `/ov:deploy` — Quadlet generation, deploy.yml, tunnel configuration
- `/ov:config` — `--sidecar` and `--list-sidecars` flags
- `/ov:secrets` — `ov secrets gpg set TS_AUTHKEY` for auth key storage
- `/ov-images:selkies-desktop` — Full deployment example with Tailscale exit node

## Source

`ov/sidecar.go` (types, merge, resolution), `ov/sidecar.yml` (embedded templates), `ov/quadlet_pod.go` (pod + sidecar quadlet generation).
