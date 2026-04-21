---
name: bootc-config
description: |
  Bootc system configuration: tty1 autologin, graphical target, pipewire/wireplumber enablement,
  and the systemd-user supervisord autostart unit that brings up supervisord-managed desktop
  services on bootc. Canonical home for any bootc-side boot wiring.
  Use when working with bootc images, autologin, systemd graphical target, or the
  supervisord-under-systemd autostart pattern.
---

# bootc-config -- bootc system configuration

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `tasks:`, `layer.yml` |

## Packages

- `sway-systemd` (RPM) -- Sway systemd integration

## What this layer configures

1. **tty1 autologin** — writes `/etc/systemd/system/getty@tty1.service.d/autologin.conf` so the `user` account logs in automatically on the first virtual console.
2. **Default graphical target** — `systemctl set-default graphical.target` so the OS boots into the desktop, not multi-user.
3. **Global PipeWire + WirePlumber** — enables `pipewire.socket` and `wireplumber.service` for every user session via `systemctl --global enable`.
4. **Systemd user supervisord autostart** (see next section) — ships `/etc/systemd/user/supervisord.service` + `systemctl --global enable supervisord.service`.
5. **Linger for `user`** — writes the sentinel file `/var/lib/systemd/linger/user` so the user's systemd instance starts at boot even without an interactive login.

Each `systemctl` cmd is suffixed `|| true` because the directives succeed on live systems but may fail harmlessly during offline bootc assembly.

## Running supervisord under systemd (bootc desktop autostart)

The selkies-desktop and openclaw metalayers compose supervisord `service:` fragments for every desktop program (labwc, chrome, swaync, waybar, selkies, traefik, …). On **container** images that's fine: `ENTRYPOINT=supervisord` starts the whole tree as PID 1. On **bootc** images, systemd is PID 1 and nothing wraps supervisord by default — the desktop would never come up.

This layer ships the missing wiring: a systemd **user** unit at `/etc/systemd/user/supervisord.service` that runs `/usr/bin/supervisord -c /etc/supervisord.conf -n`, guarded by `ConditionFileIsExecutable=/usr/bin/supervisord` + `ConditionPathExists=/etc/supervisord.conf` so it no-ops on bootc images that don't use supervisord. It's `systemctl --global enable`d so every user session started via tty1 autologin brings the desktop tier up.

### `StandardOutput=file:/tmp/supervisord-stdout.log` is load-bearing

The unit declares:

```
StandardOutput=file:/tmp/supervisord-stdout.log
StandardError=file:/tmp/supervisord-stderr.log
```

These aren't for log collection — they're **required** for supervisord's per-program `stdout_logfile=/dev/fd/1` entries to resolve. Under a systemd user service without this redirect, fd 1 points at the journal pipe; per-program dispatchers call `open("/dev/fd/1")` on that pipe and get `ENXIO: No such device or address`, causing every program to enter FATAL with the message `unknown error making dispatchers for <name>: ENXIO`. Redirecting stdout to a real file makes `/dev/fd/1` openable. Also: the supervisord header template was separately changed from `logfile=/dev/stdout` to `logfile=/tmp/supervisord.log` for the same reason — see `/ov-layers:supervisord`.

### Linger sentinel file trick

`loginctl enable-linger user` is the usual way to enable linger, but it requires a running logind — which the offline bootc builder doesn't have. The equivalent effect is writing an empty file at `/var/lib/systemd/linger/user`. Existence alone enables linger at boot. This layer uses a `mkdir:` + `cmd: touch` pair to plant the sentinel (empty content doesn't pass `write:` validation).

## Usage

```yaml
# image.yml — this layer is included transitively via bootc-base
my-bootc-image:
  base: "quay.io/fedora/fedora-bootc:43"
  bootc: true
  distro: ["fedora:43", fedora]   # external bases need this — see /ov:image
  layers:
    - bootc-base
    - <other desktop layers>
```

## Used In Images

Part of the `/ov-layers:bootc-base` composition layer. Used transitively in every bootc image:

- `/ov-images:selkies-desktop-bootc` — **canonical worked example** (exercises every piece of the supervisord + tty1 autologin + graphical target flow end-to-end)
- `/ov-images:openclaw-browser-bootc`
- `/ov-images:bazzite-ai`
- `/ov-images:aurora`

## Related Layers

- `/ov-layers:bootc-base` -- composition that includes this layer + sshd + qemu-guest-agent
- `/ov-layers:sshd` -- SSH server (also in bootc-base; NOPASSWD-sudo test handles the dual USER=root/1000 context)
- `/ov-layers:qemu-guest-agent` -- QEMU agent (also in bootc-base)
- `/ov-layers:supervisord` -- the init system that this layer wires into systemd on bootc
- `/ov-layers:selkies-desktop` -- the 19-sublayer desktop metalayer that this autostart brings up
- `/ov-images:selkies-desktop-bootc` -- worked example

## Related Skills

- `/ov:vm` -- VM lifecycle + the `/dev:/dev` mount requirement for `bootc install to-disk`
- `/ov:image` -- the external-base `distro:` requirement
- `/ov:generate` -- the empty-`systemd-services`-stage fix that makes bootc images with only packaged systemd units (unified `service:` entries using `use_packaged:`) build cleanly
- `/ov:test` -- dual-mode USER context authoring gotcha (test #11)
- `/ov:layer` -- layer authoring reference

## When to Use This Skill

Use when the user asks about:

- Bootc image autologin configuration
- systemd graphical target setup
- pipewire/wireplumber system service enablement
- tty1 autologin in bootable containers
- How supervisord starts on a bootc image (systemd user service + `StandardOutput=file:` trick)
- The `/var/lib/systemd/linger/user` sentinel-file pattern
- Symptoms like `unknown error making dispatchers for <name>: ENXIO` in supervisord logs
