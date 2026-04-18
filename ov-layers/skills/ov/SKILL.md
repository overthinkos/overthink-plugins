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
| Install files | `root.yml`, `bin/ov` |

## What It Provides

The `ov` binary inside containers serves two purposes:

1. **Native D-Bus agent** — `ov dbus` commands on the host delegate to the in-container binary via `engine exec container ov dbus <cmd> . <args>`. The in-container ov connects to the local D-Bus session bus using `godbus/dbus/v5` (pure Go, no external tools needed). This is the primary path for `ov dbus notify`, `ov dbus call`, `ov dbus list`, and `ov dbus introspect`.

2. **In-container CLI** — full ov functionality available inside the container for scripting, service management, and automation.

## Updating the Binary

The binary at `layers/ov/bin/ov` must be updated when the host-side ov is rebuilt with new features:

```bash
cd ov && go build -o ../bin/ov .                    # Build host binary
cp bin/ov layers/ov/bin/ov                           # Update layer binary
ov image build <image>                                     # Rebuild affected images
```

## `ov status` Probe

The `ov` probe checks:
1. Whether the `ov` binary exists in the container
2. The CalVer version (`ov version`)

Shows as `ov:ok (2026.94.1417)` in `ov status` detail view. Returns `-` for images without the `ov` layer.

**Note:** `ov version` uses Go's `println()` which writes to stderr. The probe uses `CombinedOutput()` to capture it correctly.

## Usage

```yaml
# images.yml -- now included in all images with supervisord
my-image:
  layers:
    - ov
```

## Used In Images

- Part of the `ov-full` composition layer (githubrunner, fedora-ov, arch-ov)
- Now directly added to all images with supervisord (openclaw, jupyter, ollama, sway-browser-vnc, selkies-desktop, immich, etc.)

## Related Layers

- `/ov-layers:ov-full` -- composition that includes ov + virtualization + gocryptfs + socat
- `/ov-layers:dbus` -- D-Bus session bus (ov dbus commands need this)
- `/ov-layers:swaync` -- notification daemon (needed for ov dbus notify to show popups)

## When to Use This Skill

Use when the user asks about:

- Installing the ov binary inside containers
- The ov-full composition layer
- In-container ov CLI usage
- Native D-Bus support (ov dbus commands delegate to in-container binary)
- Updating the ov layer binary after code changes
