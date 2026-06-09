---
name: tailscale
description: |
  Tailscale mesh VPN (tailscaled service).
  Installs the tailscale package from upstream, enables tailscaled.service via systemd.
  Use when adding Tailscale as a standalone systemd service to an image — distinct from the deploy-time Tailscale tunnel/sidecar model.
---

# tailscale -- Tailscale mesh VPN daemon

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | none |
| Ports | none (WireGuard uses UDP; no host port mapping) |
| Service | `tailscaled.service` (systemd, enabled at build time) |
| Install files | `task:` |

## Packages

- Fedora / RHEL (`rpm:`): `tailscale` from the upstream `tailscale-stable` repo (`https://pkgs.tailscale.com/stable/fedora/tailscale.repo`)
- Arch Linux (`pac:`): `tailscale`

## Usage

```yaml
# box.yml -- typical bootc composition
my-bootc-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  distro: ["fedora:43", fedora]
  layers:
    - bootc-base
    - tailscale
    - ...
```

The layer's `cmd:` task issues `systemctl enable tailscaled.service` at build time (suffixed with `|| true` because offline bootc assembly can't fully activate a live systemd — same `|| true` pattern used in `/charly-distros:bootc-config` for `systemctl set-default graphical.target`).

## Runtime activation

The image does **not** bring up the mesh on first boot — `tailscale up --authkey=tskey-…` is a runtime concern, not a build-time concern. Options:

- Interactive SSH after boot: `sudo tailscale up` and copy the login URL.
- Auth key via cloud-init or a systemd drop-in that reads a secret from `/etc/tailscale/authkey` (out of scope for this layer).

For `target: local` host deploys (canonical: `local.charly-cachyos`), pair this layer with `/charly-infrastructure:tailscale-up` — the runtime-config sibling that sets `--operator=$account` so non-root user-systemd quadlets can run `tailscale serve` (the per-pod `tunnel: tailscale` mechanism in `deploy.yml`), and that keeps the tailnet device name in sync with `hostname -s` across hostname changes. `tailscale-up` self-gates on `systemctl is-active tailscaled` so it's a no-op in image-build / pre-auth contexts; bootc consumers don't include it.

## Used In Images

- Available to any bootc/VM image that wants its own tailnet identity baked in at boot (no enabled image currently composes it).

## Tests

Two declarative checks (build-scope):

- `tailscale-binary` — `/usr/bin/tailscale` and `/usr/sbin/tailscaled` executables exist
- `tailscaled-unit-enabled` — `systemctl is-enabled tailscaled.service` returns `enabled`

## Relationship to the other two places tailscale lives in this repo

1. **This layer** (`/charly-infrastructure:tailscale`) — bakes the **daemon** into a system image as a first-class systemd service. The image runs its own tailnet node. Use for bootc/VM images.
2. **`/charly-distros:container-nesting`** — also installs the tailscale package, but as a **tool** inside a container-in-container harness (rootless podman with Tailscale-backed outbound). Different use case; don't use both in the same image.
3. **Deploy-mode tunnel/sidecar** (`/charly-automation:sidecar`, `/charly-core:deploy`) — a separate deployment-time decision that runs tailscale in a **sidecar container** alongside your app pod, giving the app a tailnet identity without baking the daemon into the app image. This is `deploy.yml`-only state and is not affected by whether this layer is present.

All three can coexist, but for most cases you want exactly one.

## Related Skills

- `/charly-infrastructure:tailscale-up` — runtime-config sibling for `target: local` host deploys (sets `--operator` + `--hostname`). Use both layers together on host targets that need `tailscale serve` to work without sudo.
- `/charly-distros:container-nesting` — the previous home of the tailscale package (bundled with buildah/skopeo/docker for nested podman; separate concern)
- `/charly-distros:bootc-config` — companion layer for bootc boot wiring (autologin, graphical target, supervisord user service)
- `/charly-automation:sidecar` — deploy-time Tailscale sidecar pattern (alternative, not a replacement)
- `/charly-core:deploy` — `deploy.yml` tunnel/sidecar configuration
- `/charly-image:layer` — layer authoring reference
- `/charly-eval:eval` — declarative testing reference

## When to Use This Skill

Use when the user asks about:

- Baking the Tailscale daemon into a bootc or container image
- The difference between this layer and the sidecar model
- Why `tailscaled.service` appears in a bootc image's service list
- First-boot activation of tailscale
