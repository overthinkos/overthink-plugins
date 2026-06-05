---
name: keepassxc
description: |
  KeePassXC password manager desktop app.
  Single-responsibility layer (rpm+pac, one test).
  Use when adding KeePassXC to an image as a standalone layer rather than pulling in the broader desktop-apps grab-bag.
---

# keepassxc -- KeePassXC password manager

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | none |
| Ports | none |
| Service | none (on-demand GUI) |
| Install files | `task:` only |

## Packages

- Fedora / RHEL (`rpm:`): `keepassxc`
- Arch Linux (`pac:`): `keepassxc`

Single package, no custom repos. Ships the KeePassXC GUI binary at `/usr/bin/keepassxc`.

## Usage

```yaml
# box.yml
my-desktop-image:
  layers:
    - selkies-desktop      # or another desktop base
    - keepassxc            # this layer
```

## Used In Images

- `/ov-distros:bazzite` — bootc desktop image that bundles KeePassXC via the `/ov-selkies:desktop-apps` layer set

## Tests

One declarative check (build-scope):

- `keepassxc-binary` — `/usr/bin/keepassxc` executable exists

No deploy-scope tests: KeePassXC is a GUI app launched on-demand by the user inside the desktop session — there's no listening port or headless responder to probe.

## Relationship to `/ov-build:secrets`

This layer is the **GUI** for editing `.kdbx` databases. `/ov-build:secrets` is the ov CLI credential store — which talks to the **Secret Service** (system keyring), NOT to a `.kdbx` file directly.

- Author a `.kdbx` file in KeePassXC (GUI) → enable its **FdoSecrets** plugin (Settings → Secret Service Integration) and mark the group "Secret Service exposed" → its entries appear on the Secret Service bus, where `ov`'s keyring backend reads them. No `ov`-side `.kdbx` configuration is involved.
- See `/ov-infrastructure:keepassxc-keyring` for the layer that wires KeePassXC as the host's Secret Service provider.

## Alternative: bundled placement in `desktop-apps`

`keepassxc` also ships inside `/ov-selkies:desktop-apps` alongside `btop`, `chromium`, `cockpit`, `transmission`, `vlc`, `zsh` — the right choice when you want the whole bundle. Use this single-responsibility layer when you want KeePassXC without dragging in the rest.

## Related Skills

- `/ov-selkies:desktop-apps` — bundle that also includes keepassxc (use when you want the full desktop-app set)
- `/ov-distros:bazzite` — bootc desktop image whose `desktop-apps` set is the primary consumer of KeePassXC
- `/ov-build:secrets` — ov CLI credential store (Secret Service + GPG; reads a KeePassXC database only via its FdoSecrets / Secret Service exposure)
- `/ov-image:layer` — layer authoring reference
- `/ov-eval:eval` — declarative testing reference

## When to Use This Skill

Use when the user asks about:

- Adding KeePassXC to a container or VM image without pulling in `desktop-apps`
- Why keepassxc lives in two places (standalone layer vs. desktop-apps bundle)
- Relationship between the KeePassXC GUI and `ov secrets`
