---
name: ov
description: |
  Overthink CLI (ov) binary installed into container/VM images for in-container use.
  Use when working with ov binary deployment inside containers, native D-Bus support, or the ov-full composition.
---

# ov -- Overthink CLI binary

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `tasks:`, `bin/ov` |

## What It Provides

The `ov` binary inside containers serves two purposes:

1. **Native D-Bus agent** — `ov eval dbus` commands on the host delegate to the in-container binary via `engine exec container ov eval dbus <cmd> . <args>`. The in-container ov connects to the local D-Bus session bus using `godbus/dbus/v5` (pure Go, no external tools needed). This is the primary path for `ov eval dbus notify`, `ov eval dbus call`, `ov eval dbus list`, and `ov eval dbus introspect`.

2. **In-container CLI** — full ov functionality available inside the container for scripting, service management, and automation.

## Updating the Binary — dual-path gotcha

The `ov` layer's `copy: bin/ov` task is resolved **relative to the layer directory**, so the image build reads `layers/ov/bin/ov` — NOT the repo-root `bin/ov`. Two independent paths need to stay in sync:

| Path | Who reads it |
|------|-------------|
| `bin/ov` (repo root) | Host-side `ov` invocations; users running `/tmp/ov` style tests |
| `layers/ov/bin/ov` | The `ov` layer's COPY into images during `ov image build` |

**Canonical workflow** — `task build:ov` compiles to repo-root AND syncs to the layer:

```bash
task build:ov                              # Builds + syncs both paths; rebuild images after.
ov image build <image>                     # Rebuild affected images.
```

**Manual workflow** — if you skip `task build:ov` and build with `go build` directly, you MUST sync the layer path, or images will bake the previous binary:

```bash
cd ov && go build -o ../bin/ov .           # Only updates repo-root bin/ov.
cp bin/ov layers/ov/bin/ov                 # REQUIRED — sync to layer path.
ov image build <image>                     # Rebuild affected images.
```

**Why this bites**: `ov image build` uses auto-generated intermediate images (e.g., `ghcr.io/overthinkos/fedora-ov-2-dbus-nodejs`) that cache the `ov` layer. If you update `bin/ov` in repo-root but forget the layer copy, the intermediate's cache hit serves stale content. After cleaning up a stale dual-path situation, `podman rmi 'ghcr.io/overthinkos/fedora-ov-2*'` forces a clean intermediate rebuild.

## `ov status` Probe

The `ov` probe checks:
1. Whether the `ov` binary exists in the container
2. The CalVer version (`ov version`)

Shows as `ov:ok (2026.94.1417)` in `ov status` detail view. Returns `-` for images without the `ov` layer.

**Note:** `ov version` writes to **stdout** via `fmt.Println` (the prior
`println(version)` emitted to stderr; the move to `fmt.Println` landed
with the MCP server work so the in-process tool-call path — which
captures `os.Stdout` — returns the CalVer correctly). The layer test at
`layers/ov/layer.yml` asserts `stdout:` matches `[0-9]{4}\.[0-9]+`. The
`ov status` probe uses `CombinedOutput()` so it's agnostic to the
stream.

## Usage

```yaml
# image.yml -- now included in all images with supervisord
my-image:
  layers:
    - ov
```

## Used In Images

- Part of the `ov-full` composition layer (githubrunner, fedora-ov, arch-ov)
- Now directly added to all images with supervisord (openclaw, jupyter, ollama, sway-browser-vnc, selkies-desktop, immich, etc.)

## Related Layers

- `/ov-layers:ov-full` -- composition that includes ov + virtualization + gocryptfs + socat
- `/ov-layers:ov-mcp` -- layers: [ov, supervisord] meta-composition that deploys `ov mcp serve` (~192-tool MCP gateway) with a `/workspace` bind mount (volume NAME `project`) for build-mode tools + auto-fallback to overthinkos/overthink when nothing is bound
- `/ov-layers:dbus` -- D-Bus session bus (ov eval dbus commands need this)
- `/ov-layers:swaync` -- notification daemon (needed for ov eval dbus notify to show popups)

## When to Use This Skill

Use when the user asks about:

- Installing the ov binary inside containers
- The ov-full composition layer
- In-container ov CLI usage
- Native D-Bus support (ov eval dbus commands delegate to in-container binary)
- Updating the ov layer binary after code changes

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
