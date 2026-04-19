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

1. **Native D-Bus agent** — `ov test dbus` commands on the host delegate to the in-container binary via `engine exec container ov test dbus <cmd> . <args>`. The in-container ov connects to the local D-Bus session bus using `godbus/dbus/v5` (pure Go, no external tools needed). This is the primary path for `ov test dbus notify`, `ov test dbus call`, `ov test dbus list`, and `ov test dbus introspect`.

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

**Note:** `ov version` uses Go's `println()` which writes to stderr. The probe uses `CombinedOutput()` to capture it correctly.

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
- `/ov-layers:dbus` -- D-Bus session bus (ov test dbus commands need this)
- `/ov-layers:swaync` -- notification daemon (needed for ov test dbus notify to show popups)

## When to Use This Skill

Use when the user asks about:

- Installing the ov binary inside containers
- The ov-full composition layer
- In-container ov CLI usage
- Native D-Bus support (ov test dbus commands delegate to in-container binary)
- Updating the ov layer binary after code changes
