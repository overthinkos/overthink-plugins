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
| Install files | `tasks:` |

## Packages

- Fedora / RHEL (`rpm:`): `tailscale` from the upstream `tailscale-stable` repo (`https://pkgs.tailscale.com/stable/fedora/tailscale.repo`)
- Arch Linux (`pac:`): `tailscale`

## Usage

```yaml
# image.yml -- typical bootc composition
my-bootc-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  distro: ["fedora:43", fedora]
  layers:
    - bootc-base
    - tailscale
    - ...
```

The layer's `cmd:` task issues `systemctl enable tailscaled.service` at build time (suffixed with `|| true` because offline bootc assembly can't fully activate a live systemd — same `|| true` pattern used in `/ov-layers:bootc-config` for `systemctl set-default graphical.target`).

## Runtime activation

The image does **not** bring up the mesh on first boot — `tailscale up --authkey=tskey-…` is a runtime concern, not a build-time concern. Options:

- Interactive SSH after boot: `sudo tailscale up` and copy the login URL.
- Auth key via cloud-init or a systemd drop-in that reads a secret from `/etc/tailscale/authkey` (out of scope for this layer).

## Used In Images

- `/ov-images:selkies-desktop-bootc` — primary consumer (bootc VM that wants a tailnet identity at boot).

## Tests

Two declarative checks (build-scope):

- `tailscale-binary` — `/usr/bin/tailscale` and `/usr/sbin/tailscaled` executables exist
- `tailscaled-unit-enabled` — `systemctl is-enabled tailscaled.service` returns `enabled`

## Relationship to the other two places tailscale lives in this repo

1. **This layer** (`/ov-layers:tailscale`) — bakes the **daemon** into a system image as a first-class systemd service. The image runs its own tailnet node. Use for bootc/VM images.
2. **`/ov-layers:container-nesting`** — also installs the tailscale package, but as a **tool** inside a container-in-container harness (rootless podman with Tailscale-backed outbound). Different use case; don't use both in the same image.
3. **Deploy-mode tunnel/sidecar** (`/ov:sidecar`, `/ov:deploy`) — a separate deployment-time decision that runs tailscale in a **sidecar container** alongside your app pod, giving the app a tailnet identity without baking the daemon into the app image. This is `deploy.yml`-only state and is not affected by whether this layer is present.

All three can coexist, but for most cases you want exactly one.

## Related Skills

- `/ov-layers:container-nesting` — the previous home of the tailscale package (bundled with buildah/skopeo/docker for nested podman; separate concern)
- `/ov-images:selkies-desktop-bootc` — primary consumer
- `/ov-layers:bootc-config` — companion layer for bootc boot wiring (autologin, graphical target, supervisord user service)
- `/ov:sidecar` — deploy-time Tailscale sidecar pattern (alternative, not a replacement)
- `/ov:deploy` — `deploy.yml` tunnel/sidecar configuration
- `/ov:layer` — layer authoring reference
- `/ov:test` — declarative testing reference

## When to Use This Skill

Use when the user asks about:

- Baking the Tailscale daemon into a bootc or container image
- The difference between this layer and the sidecar model
- Why `tailscaled.service` appears in a bootc image's service list
- First-boot activation of tailscale
