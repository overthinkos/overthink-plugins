---
name: eval-sway-browser-vnc
description: |
  The DISPOSABLE pod bed (eval-sway-browser-vnc-pod) for R10 testing of the Sway
  desktop verb surface (cdp / wl / vnc / dbus / mcp / record). Deploys the
  shipping sway-browser-vnc image. Use when running or maintaining the
  live-container eval bed for the sway stack.
---

# eval-sway-browser-vnc

`eval-sway-browser-vnc-pod` is the canonical **disposable** pod bed for the
live-container test verbs (cdp / wl / vnc / dbus / mcp / record) on the Sway
stack. It deploys the **shipping `/ov-selkies:sway-browser-vnc` image directly**
— there is no separate eval image. It sits alongside the other `eval-*` smoke
beds (`eval-pod` — the combined image/layer/pod/DeployTarget mechanism bed —
and `eval-local`) — all `kind: eval` entities in this repo's `eval.yml`.

## Bed (`eval.yml`)

```yaml
eval:
  eval-sway-browser-vnc-pod:
    target: pod
    image: sway-browser-vnc        # the shipping image, deployed as-is
    disposable: true
    lifecycle: dev
    eval:                          # delta probes sway-browser-vnc doesn't bake
      - { http: "http://127.0.0.1:${HOST_PORT:9222}/json/version", id: esbv-pod-http-cdp, scope: deploy, status: 200 }
      - { cdp: list,        id: esbv-pod-cdp-list,     scope: deploy }
      - { wl: sway-tree,    id: esbv-pod-wl-sway-tree, scope: deploy }
      - { record: start,    id: esbv-pod-record-start, scope: deploy, record_name: eval-term, record_mode: terminal }
```

`disposable: true` is the sole authorization for `ov update`/`ov remove` to
destroy + rebuild the bed unattended (see `/ov-internals:disposable`). The bed
publishes `sway-browser-vnc`'s canonical ports `5900/9222/9224`, so it shares
those host ports with a real `sway-browser-vnc` deployment — only one runs at a
time (`ov image validate` notes this).

## Probe coverage

`sway-browser-vnc` already bakes binaries/services + cdp/vnc/wl/dbus checks, and
inherits the two `mcp:` probes from the `chrome-devtools-mcp` layer. The bed's
`eval:` block above adds the remaining deploy-scope probes — operator-side
`http:` (CDP `/json/version` via `HOST_PORT`), `cdp: list`, `wl: sway-tree`, and
`record: start` — so a single `ov eval live` run exercises the full
cdp/wl/vnc/dbus/mcp/record surface.

## Usage

```bash
# Canonical one-shot — the FULL R10 acceptance sequence (build → eval image →
# deploy → config → start → eval live → fresh update → tear down):
ov eval run eval-sway-browser-vnc-pod
# Or drive the steps manually:
ov config eval-sway-browser-vnc-pod
ov start  eval-sway-browser-vnc-pod
ov eval live eval-sway-browser-vnc-pod        # deploy-scope cdp/wl/vnc/dbus/mcp/record
ov update eval-sway-browser-vnc-pod && ov eval live eval-sway-browser-vnc-pod  # fresh-rebuild re-verify
```

Note: `ov config <key>` persists `image: <key>` (it assumes deploy-key ==
image-name; see `/ov-core:deploy`). Because this bed's key
(`eval-sway-browser-vnc-pod`) differs from its image (`sway-browser-vnc`), set
the operator ref to `sway-browser-vnc` in `~/.config/ov/deploy.yml` (the eval
runner / `ov deploy add` does this for you — `ov` "never clobbers
operator-authored refs").

## Related Skills

- `/ov-selkies:sway-browser-vnc` — the shipping image this bed deploys
- `/ov-eval:eval` — the eval framework + the disposable test-bed table
- `/ov-eval:cdp`, `/ov-eval:wl`, `/ov-eval:vnc`, `/ov-eval:dbus`, `/ov-eval:record`,
  `/ov-build:ov-mcp-cmd` — the live-container verbs this bed covers
- `/ov-core:deploy` — `disposable:` bed semantics, the deploy-key vs image-name caveat
- `/ov-internals:disposable` — the `disposable: true` authorization model

## When to Use This Skill

**MUST be invoked** when deploying, running, or troubleshooting the
`eval-sway-browser-vnc-pod` bed, or when running R10 verification for the Sway
desktop verb surface.
