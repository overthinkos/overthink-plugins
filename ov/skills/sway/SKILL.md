---
name: sway
description: |
  DEPRECATED: Sway commands have moved to `ov wl sway <subcommand>`. See `/ov:wl` for the unified desktop automation command.
---

# Sway - DEPRECATED (moved to `ov wl sway`)

## Notice

**`ov sway` has been merged into `ov wl sway`.**

All sway-specific commands now live under `ov wl sway <subcommand>`:

```bash
# Old (no longer works):
ov sway tree my-app
ov sway msg my-app 'focus left'

# New:
ov wl sway tree my-app
ov wl sway msg my-app 'focus left'
```

Compositor-agnostic commands (focus, close, fullscreen, minimize, exec, resolution) are top-level `ov wl` subcommands that work on both sway and labwc.

## See `/ov:wl` for the full reference.
