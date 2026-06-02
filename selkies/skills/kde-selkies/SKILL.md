---
name: kde-selkies
description: >
  The KDE nested-compositor PRIMITIVE for the selkies streaming desktop —
  startplasma-wayland nested in pixelflux, de-SDDM, started by a supervisord
  poll-for-wayland-1 service. MUST be invoked before editing the kde-selkies
  layer or its kde-selkies-session wrapper.
---

# kde-selkies — KDE Plasma nested-compositor primitive (the KDE seam)

`kde-selkies` is the KDE analogue of the `labwc` layer: the swappable
nested-compositor primitive that renders a desktop into pixelflux's `wayland-1`.
It runs a FULL KDE Plasma Wayland session (`startplasma-wayland` =
kwin_wayland + plasmashell) nested in pixelflux, so selkies streams Plasma.

## Headless de-SDDM design (load-bearing)

`require: [selkies, kde-shell, pipewire, dbus]` — NOT `kde-desktop`. A pod has
no DRM seat, no SDDM, no `graphical.target`. So:

- ONE supervisord service `kde-selkies-session` (priority 12, `scope: user`,
  `%(ENV_HOME)s` exec — resolves for both supervisord pods AND systemd-user
  targets via `service_render.go`). **No `after: graphical-session.target`** —
  the wrapper's poll-for-`/tmp/wayland-1` IS the ordering primitive (identical to
  `labwc-wrapper`).
- `kde-selkies-session` waits for pixelflux's `wayland-1`, sets
  `WAYLAND_DISPLAY=wayland-1` (so kwin renders INTO pixelflux), then
  `exec dbus-run-session startplasma-wayland`. kwin creates `wayland-0` for
  Plasma's own clients.
- **Chrome autostart:** the script ALSO backgrounds a poll-for-`wayland-0`
  launcher that runs `chrome-wrapper` with `WAYLAND_DISPLAY=wayland-0` once kwin
  publishes the client socket — the KDE analogue of labwc's autostart. Chrome is
  NOT a supervisord program (compositor-launched, same as labwc); without this
  launcher `cdp-proxy` listens but Chrome's CDP backend is dead (`/json/version`
  → EOF). Chrome works headless under KDE once launched into `wayland-0`.

This headless-no-seat path is PROVEN on a real pod (`eval-selkies-kde-pod`,
97/97): `kde-selkies-session` RUNNING ≥20s, `https://:3000/` → 200, Chrome CDP
`/json/version` → 200.

## Encoder

`kde-selkies` sets NO `SELKIES_ENCODER` — pixelflux auto-detects at runtime
(VAAPI on an AMD/Intel renderD via libva-native; NVENC from the cuda-arch-builder
pixelflux on the `*-nvidia` image; x264 otherwise). So ONE kde-selkies layer
streams on every GPU config. The encoder identity is asserted per-image in the
GPU eval beds, not here.

## Related

- `/ov-selkies:labwc` — the other nested-compositor primitive (the labwc seam).
- `/ov-selkies:kde-shell` — the SDDM-free Plasma session packages kde-selkies requires.
- `/ov-selkies:selkies-kde-desktop` — the flavor metalayer composing this.
- `/ov-distros:cachyos` — CachyOS/Arch base where the KDE Plasma stack lives.
