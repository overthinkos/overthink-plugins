---
name: check-sway-browser-vnc
description: |
  The DISPOSABLE pod bed (check-sway-browser-vnc-pod) for R10 testing of the Sway
  desktop verb surface (cdp / wl / vnc / dbus / mcp / record). Deploys the
  shipping sway-browser-vnc image. Use when running or maintaining the
  live-container check bed for the sway stack.
---

# check-sway-browser-vnc

`check-sway-browser-vnc-pod` is the canonical **disposable** pod bed for the
live-container test verbs (cdp / wl / vnc / dbus / mcp / record) on the Sway
stack. It deploys the **shipping `/charly-selkies:sway-browser-vnc` image directly**
— there is no separate check image. It sits alongside the other `check-*` smoke
beds (`check-pod` — the combined image/layer/pod/DeployTarget mechanism bed —
and `check-local`) — all `kind: check` entities in their project's `charly.yml`
`check:` block (this bed and `check-pod` in `box/fedora`; `check-local` in main).

## Bed (a `kind: check` entity in the `check:` block)

```yaml
check:
  check-sway-browser-vnc-pod:
    target: pod
    box: sway-browser-vnc        # the shipping image, deployed as-is
    disposable: true
    lifecycle: dev
    scenario:                      # delta probes sway-browser-vnc doesn't bake
      - name: sway-desktop-verb-surface
        step:                      # each step is one inline Op; a probe verb defaults to do: assert
          - then: CDP /json/version answers over the published HOST_PORT
            http: "http://127.0.0.1:${HOST_PORT:9222}/json/version"
            id: esbv-pod-http-cdp
            context: [deploy]
            status: 200
          - then: CDP enumerates the live debugging targets
            cdp: list
            id: esbv-pod-cdp-list
            context: [deploy]
          - then: the Sway tree is reachable over the Wayland socket
            wl: sway-tree
            id: esbv-pod-wl-sway-tree
            context: [deploy]
          - then: a terminal recording starts
            record: start
            id: esbv-pod-record-start
            context: [deploy]
            record_name: check-term
            record_mode: terminal
```

`disposable: true` is the sole authorization for `charly update`/`charly remove` to
destroy + rebuild the bed unattended (see `/charly-internals:disposable`). The bed
publishes `sway-browser-vnc`'s canonical ports `5900/9222/9224`, so it shares
those host ports with a real `sway-browser-vnc` deployment — only one runs at a
time (`charly box validate` notes this).

## Probe coverage

`sway-browser-vnc` already bakes binaries/services + cdp/vnc/wl/dbus scenario
steps, and inherits the two `mcp:` probes from the `chrome-devtools-mcp` layer.
The bed's `scenario:` block above adds the remaining deploy-context steps —
operator-side `http:` (CDP `/json/version` via `HOST_PORT`), `cdp: list`,
`wl: sway-tree`, and `record: start` — so a single `charly check live` run
exercises the full cdp/wl/vnc/dbus/mcp/record surface.

## Usage

```bash
# Canonical one-shot — the FULL R10 acceptance sequence (build → check image →
# deploy → config → start → check live → fresh update → tear down):
charly check run check-sway-browser-vnc-pod
# Or drive the steps manually:
charly config check-sway-browser-vnc-pod
charly start  check-sway-browser-vnc-pod
charly check live check-sway-browser-vnc-pod        # deploy-context cdp/wl/vnc/dbus/mcp/record
charly update check-sway-browser-vnc-pod && charly check live check-sway-browser-vnc-pod  # fresh-rebuild re-verify
```

Note: `charly config <key>` persists `image: <key>` (it assumes deploy-key ==
image-name; see `/charly-core:deploy`). Because this bed's key
(`check-sway-browser-vnc-pod`) differs from its image (`sway-browser-vnc`), set
the operator ref to `sway-browser-vnc` in `~/.config/charly/charly.yml` (the check
runner / `charly deploy add` does this for you — `charly` "never clobbers
operator-authored refs").

## Related Skills

- `/charly-selkies:sway-browser-vnc` — the shipping image this bed deploys
- `/charly-check:check` — the check framework + the disposable test-bed table
- `/charly-check:cdp`, `/charly-check:wl`, `/charly-check:vnc`, `/charly-check:dbus`, `/charly-check:record`,
  `/charly-build:charly-mcp-cmd` — the live-container verbs this bed covers
- `/charly-core:deploy` — `disposable:` bed semantics, the deploy-key vs image-name caveat
- `/charly-internals:disposable` — the `disposable: true` authorization model

## When to Use This Skill

**MUST be invoked** when deploying, running, or troubleshooting the
`check-sway-browser-vnc-pod` bed, or when running R10 verification for the Sway
desktop verb surface.
