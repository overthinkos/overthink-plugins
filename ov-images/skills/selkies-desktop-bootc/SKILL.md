---
name: selkies-desktop-bootc
description: |
  Bootable (bootc) VM image combining the selkies-desktop streaming desktop
  with Tailscale (mesh VPN) and KeePassXC (password manager). Fedora 43
  base. Boots under libvirt/QEMU as a full OS. Canonical worked example
  of the external-base-bootc + explicit-distro pattern.
  MUST be invoked before building, deploying, or troubleshooting selkies-desktop-bootc.
---

# selkies-desktop-bootc

Bootable container image: Fedora 43 bootc + Selkies browser-streamed desktop + Tailscale + KeePassXC. Ships as a QCOW2 disk suitable for libvirt/QEMU/bare-metal.

## Image Properties

| Property | Value |
|----------|-------|
| Base | `quay.io/fedora/fedora-bootc:43` |
| Bootc | `true` |
| Distro tags | `["fedora:43", fedora]` **(must be declared — external bases do not inherit)** |
| Layers | agent-forwarding, bootc-base, rpmfusion, selkies-desktop, tailscale, keepassxc, dbus, ov |
| Platforms | linux/amd64 |
| Ports (host → container) | 13000 → 3000, 19222 → 9222, 19224 → 9224 |
| VM ssh_port (host → VM:22) | 2250 |
| Status | **enabled by default** during this worked-example iteration; revert to `enabled: false` once shipped |
| Registry | ghcr.io/overthinkos |

## VM Configuration

| Setting | Value | Rationale |
|---------|-------|-----------|
| SSH port | 2250 | Non-default to avoid colliding with `ov-selkies-desktop*` containers that claim host :2222 |
| Disk size | 40 GiB | Selkies + Chrome + PipeWire + toolchain transitives need more than openclaw-browser-bootc's 20 GiB |
| RAM | 8 G | Chrome + compositor + recorder want headroom |
| CPUs | 4 | Matches 8 G/4 rule of thumb for a streaming desktop VM |
| Rootfs | ext4 | Default (fallback when image.yml omits `rootfs:`) |

## Full Layer Stack

1. `fedora-bootc:43` (external bootc base)
2. `/ov-layers:agent-forwarding` — SSH/GPG agent socket plumbing
3. `/ov-layers:bootc-base` — `sshd` + `qemu-guest-agent` + `/ov-layers:bootc-config`
4. `/ov-layers:rpmfusion` — free + nonfree repos (required for ffmpeg in selkies-desktop)
5. `/ov-layers:selkies-desktop` — 19-sublayer metalayer (pipewire, chrome, labwc, waybar-labwc, swaync, selkies, desktop-fonts, wl-tools, a11y-tools, xterm, tmux, asciinema, fastfetch, wl-overlay, wl-record-pixelflux, wl-screenshot-pixelflux, pavucontrol, swaync, sshd)
6. `/ov-layers:tailscale` — systemd-enabled tailscaled
7. `/ov-layers:keepassxc` — GUI password manager
8. `/ov-layers:dbus` — session bus
9. `/ov-layers:ov` — ov CLI in-image

## Ports

Host ports are intentionally shifted into 13xxx/19xxx so the VM can run **alongside** active `ov-selkies-desktop` containers that already hold 3000/9222/9224/2222.

| Host port | VM port | Service | Protocol |
|-----------|---------|---------|----------|
| 13000 | 3000 | Traefik → selkies-fileserver | HTTPS (self-signed) |
| 19222 | 9222 | cdp-proxy → Chrome DevTools | HTTP |
| 19224 | 9224 | chrome-devtools-mcp | HTTP (MCP Streamable HTTP) |
| 2250 | 22 | system `sshd` | TCP |

**There is no `2222:2222` publish on this image.** Non-bootc selkies-desktop publishes 2222 for its supervisord-managed `sshd-wrapper`; under bootc, `sshd` runs as a system systemd unit on port 22 (via `/ov-layers:sshd`'s `service:` entry with `use_packaged: sshd.service`), so 2222 is dead weight and would collide with `vm.ssh_port`. See `/ov-layers:sshd` for the dual-mode story.

## Quick Start

```bash
# 1. Build the image.
ov image build selkies-desktop-bootc

# 2. Refresh rootful podman storage (ov vm build uses sudo podman).
podman save ghcr.io/overthinkos/selkies-desktop-bootc:latest -o /tmp/sdb.tar
sudo podman load -i /tmp/sdb.tar
rm -f /tmp/sdb.tar

# 3. Build the bootc disk image (QCOW2, via bootc install to-disk).
ov vm build selkies-desktop-bootc --transport containers-storage

# 4. Create + start the VM (injects your ~/.ssh/id_*.pub via SMBIOS credentials).
ov vm create selkies-desktop-bootc

# 5. SSH into the running VM.
ssh -p 2250 root@localhost
```

Prerequisites: rootful podman reachable via passwordless `sudo` (`ov settings set engine.rootful sudo`), kernel+modules consistent (`ls /lib/modules/$(uname -r)` must exist — reboot after kernel updates), and `~/.ssh/*.pub` present.

## Verification

```bash
# Inside the VM:
systemctl --failed --no-pager          # must be empty
systemctl is-active sshd tailscaled    # both active
runuser -l user -c 'supervisorctl status'  # all programs RUNNING (chrome STOPPED is intended)

# From the host:
curl -skI https://127.0.0.1:13000/     # HTTP 200, selkies HTML
curl -sS  -X POST http://127.0.0.1:19224/mcp \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2024-11-05","capabilities":{},"clientInfo":{"name":"probe","version":"1"}}}'
# → returns serverInfo.name=chrome_devtools with a real version
```

## Test Coverage

**77 declarative checks · 0 failing · 0 skipped** under `ov eval image`. Includes binary/package/service checks for every layer plus image-level composition checks (labwc + chrome + tailscale + keepassxc executables, `bootc container lint` sanity). The sshd layer's NOPASSWD-sudo check uses a `runuser -l user` wrapper to work under bootc's `USER=root` default as well as non-bootc's `USER=1000` — see `/ov-layers:sshd`.

## Known Caveats

### 1. External bases require explicit `distro:`

`base: "quay.io/fedora/fedora-bootc:43"` is an **external** base (URL, not the name of another image in `image.yml`). Unlike internal bases, it does **not** inherit distro tags. Without `distro: ["fedora:43", fedora]`, `ov image inspect` shows `"Distro": null`, the generator skips every layer's `rpm:` install (the install_template's Phase-2 branch requires `img.DistroDef != nil`), and the final image has zero packages installed from layer `rpm:` sections. See `/ov:image` for the resolution chain. This was the first bug encountered during the build; the sibling bootc templates (`/ov-images:openclaw-browser-bootc`, `/ov-images:bazzite-ai`, `/ov-images:aurora`) have the same latent issue but haven't tripped it because they're still `enabled: false`.

### 2. `dnf5-plugins` prepend is required for URL repos

`quay.io/fedora/fedora-bootc:43` strips `dnf5-plugins` (the package that provides `dnf5 config-manager`) from the default install. The generator's install_template now prepends `dnf install -y dnf5-plugins` whenever any layer `rpm.repos:` entry has a `url:` (via the `anyRepoHasURL` template helper in `ov/format_template.go`). Without it, `/ov-layers:ffmpeg` — which adds negativo17's `fedora-multimedia` repo — fails with `Unknown argument "config-manager" for command "dnf5"`. See `/ov:generate` for the generator change.

### 3. `vm.ssh_port` is honoured end-to-end

Older `ov` binaries hardcoded the QEMU hostfwd / libvirt portForward SSH mapping to 2222. Current ov plumbs `vm.ssh_port` (from the OCI `Vm` label) through `createQemu`, `createLibvirt`, and `buildDomainXML`. If your VM creates with SSH on 2222 even though `vm.ssh_port: 2250` is set, the installed `ov` binary is stale — run `task build:ov`. See `/ov:vm`.

### 4. labwc ↔ pixelflux start-order race (live caveat)

`labwc-wrapper` (inside selkies-desktop) blocks on `/tmp/wayland-1` (pixelflux's socket) at startup, but pixelflux is itself started by `selkies` (another supervisord program). With `startsecs=2`, supervisord marks labwc RUNNING before the socket appears, then both processes exit-status-1 and restart in lockstep every ~15 s before converging. The dependent services (`selkies-fileserver`, `chrome-devtools-mcp`, `traefik`, `cdp-proxy`) run stably throughout. Does not affect container-mode selkies-desktop (different entrypoint path). See `/ov-layers:selkies-desktop` "Known bootc caveat" for fix options.

### 5. `qemu-guest-agent` is `enabled/inactive` under QEMU user-net

Expected. The agent needs a `virtio-serial` channel that ov's QEMU backend doesn't expose under user-net. Switch to the libvirt backend (`ov settings set vm.backend libvirt`) to activate it.

### 6. Desktop autostart is wired through `/ov-layers:bootc-config`

Non-bootc selkies-desktop runs supervisord as container PID 1 via `ENTRYPOINT`. Under bootc there's no such entrypoint — systemd is PID 1. The `/ov-layers:bootc-config` layer ships a systemd **user** unit `/etc/systemd/user/supervisord.service` that runs `/usr/bin/supervisord -c /etc/supervisord.conf -n` and is `systemctl --global enable`d. It needs `StandardOutput=file:/tmp/supervisord-stdout.log` so the per-program `stdout_logfile=/dev/fd/1` entries (inherited from every selkies-desktop sublayer) resolve to an openable file rather than a journal pipe (which would ENXIO). This is the bootc-side wiring for the desktop. See `/ov-layers:bootc-config`.

## Quick Start: VM port remap for your environment

The 13000/19222/19224 ports chosen here are specifically to coexist with running `ov-selkies-desktop*` container instances. If you're running the VM in isolation and prefer the standard ports, edit `image.yml`:

```yaml
selkies-desktop-bootc:
  ports:
    - "3000:3000"
    - "9222:9222"
    - "9224:9224"
  vm:
    ssh_port: 2222
```

Rebuild + redeploy the VM.

## Related Images

- `/ov-images:selkies-desktop` — non-bootc container sibling (`fedora-nonfree` base, supervisord as PID 1)
- `/ov-images:selkies-desktop-nvidia` — GPU-accelerated container sibling (`nvidia` base)
- `/ov-images:selkies-desktop-ov` — GPU-accelerated container sibling + full ov toolchain (nested rootless podman + rootless libvirt VMs). The non-bootc counterpart of this bootc image when you want the desktop to itself build images / launch pods / spawn VMs.
- `/ov-images:openclaw-browser-bootc` — sibling bootc template (disabled; latent `distro:` bug per Caveat 1)
- `/ov-images:bazzite-ai` — ublue-based bootc template
- `/ov-images:aurora` — ublue-based bootc template

## Related Skills

- `/ov-layers:selkies-desktop` — the metalayer composition (19 sublayers)
- `/ov-layers:tailscale` — mesh VPN (this image's standalone tailscaled)
- `/ov-layers:keepassxc` — password manager
- `/ov-layers:bootc-base` — sshd + guest-agent + bootc-config
- `/ov-layers:bootc-config` — bootc boot wiring (tty1 autologin, graphical target, systemd-user supervisord)
- `/ov-layers:rpmfusion` — the nonfree-codec layer needed by selkies-desktop/ffmpeg
- `/ov:vm` — VM build + lifecycle + engine.rootful modes + the `/dev:/dev` mount explanation
- `/ov:build` — container image build (also covers the stale-ov-binary and buildah-cache-mount caveats)
- `/ov:generate` — Containerfile generation (empty-systemd-services-stage fix, `anyRepoHasURL`, `export BUILD_ARCH` gotcha)
- `/ov:image` — the `distro:` field semantics + external-base caveat
- `/ov:test` — declarative testing (including the USER-context gotcha for dual-mode images)

## When to Use This Skill

**MUST be invoked** when the task involves:

- Building, deploying, booting, or troubleshooting the `selkies-desktop-bootc` image
- Authoring or debugging any other external-base bootc image (this skill is the worked-example reference)
- Understanding the interaction between `/ov-layers:bootc-config` and `/ov-layers:supervisord` on bootc
- Port remapping to avoid collision with running selkies-desktop containers
