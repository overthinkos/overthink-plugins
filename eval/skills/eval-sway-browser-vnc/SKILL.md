---
name: eval-sway-browser-vnc
description: |
  Dedicated DISPOSABLE image + pod bed for R10 testing of the Sway desktop verb
  surface (cdp / wl / vnc / dbus / mcp / record). Use when running or maintaining
  the live-container eval bed for the sway stack — NOT a shipping image.
---

# eval-sway-browser-vnc

A purpose-built, **disposable** eval artifact: an eval twin of the shipping
`/ov-selkies:sway-browser-vnc` image, used as the canonical pod bed for the
live-container test verbs (cdp / wl / vnc / dbus / mcp / record). It exists so
R10 verification of the Sway surface runs on a throwaway target instead of being
coupled to a user-facing image — mirroring the other `eval-*-pod` smoke beds
(`eval-image-pod`, `eval-layer-pod`, `eval-pod-pod`, `eval-deploy-pod`,
`eval-local-deploy`).

## Image (`image.yml`)

| Property | Value |
|----------|-------|
| Base | `fedora` |
| Layers | `agent-forwarding`, `sway-desktop-vnc`, `dbus`, `ov` (the eval twin of `sway-browser-vnc`) |
| Ports (host→container) | 15900→5900 (wayvnc), 19222→9222 (Chrome CDP), 19224→9224 (chrome-devtools-mcp) — isolated host ports so the disposable bed coexists with production desktops on a shared host |
| enabled | `true` |
| Registry | ghcr.io/overthinkos |

Because it carries the **same layer list** as `sway-browser-vnc`, both images
share the auto-intermediate covering that chain, so the marginal build cost of the
eval twin is ~zero once `sway-browser-vnc` is built. The full Sway layer stack
(sway, chrome-sway, xdg-portal, xfce4-terminal, thunar, wl-screenshot-grim,
swaync, waybar, pipewire, wl-*, wf-recorder, fastfetch, asciinema, tmux) resolves
transitively through `sway-desktop-vnc → sway-desktop`.

The image carries a baked `eval:` block (binary + service health, cdp / wl / vnc /
dbus probes) so the probes ship with the image; the bed adds the operator-side and
mcp / record deploy-scope probes.

## Bed (`deploy.yml`)

```yaml
deploy:
  eval-sway-browser-vnc-pod:
    target: pod
    image: eval-sway-browser-vnc
    disposable: true
    lifecycle: dev
    # No port: / eval: block — the image declares the isolated host ports
    # (15900/19222/19224) and carries the baked deploy-scope eval probes.
```

`disposable: true` is the sole authorization for `ov update` to destroy + rebuild
the bed unattended (see `/ov-internals:disposable`). There is **no** openclaw
gateway here — the bed does not probe port 18789.

## Usage

```bash
# Build the disposable image + run its baked checks (build-scope):
ov image build eval-sway-browser-vnc
ov eval image eval-sway-browser-vnc

# Full live R10 against the disposable pod bed (deploy-scope: cdp/wl/vnc/dbus/mcp/record):
ov update eval-sway-browser-vnc-pod          # destroy → rebuild → start (disposable)
ov eval live eval-sway-browser-vnc-pod
```

## Relationship to `sway-browser-vnc`

`/ov-selkies:sway-browser-vnc` is the single **shipping** Sway+VNC+Chrome image.
`eval-sway-browser-vnc` is its disposable eval twin — same composition, extra
baked probes, deployed only as a throwaway R10 bed. When the Sway desktop layers
change, this is the bed that exercises the live verb surface end-to-end.

## Related Skills

- `/ov-selkies:sway-browser-vnc` — the shipping image this twins
- `/ov-eval:eval` — the eval framework + the disposable test-bed table
- `/ov-eval:cdp`, `/ov-eval:wl`, `/ov-eval:vnc`, `/ov-eval:dbus`, `/ov-eval:record`,
  `/ov-build:ov-mcp-cmd` — the live-container verbs this bed covers
- `/ov-core:deploy` — `disposable:` bed semantics + `ov update`
- `/ov-internals:disposable` — the `disposable: true` authorization model

## When to Use This Skill

**MUST be invoked** when building, deploying, or troubleshooting the
`eval-sway-browser-vnc` image / `eval-sway-browser-vnc-pod` bed, or when running
R10 verification for the Sway desktop verb surface.
