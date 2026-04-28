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
| Install files | `tasks:` only |

## Packages

- Fedora / RHEL (`rpm:`): `keepassxc`
- Arch Linux (`pac:`): `keepassxc`

Single package, no custom repos. Ships the KeePassXC GUI binary at `/usr/bin/keepassxc`.

## Usage

```yaml
# image.yml
my-desktop-image:
  layers:
    - selkies-desktop      # or another desktop base
    - keepassxc            # this layer
```

## Used In Images

- `/ov-images:selkies-desktop-bootc` — Fedora bootc VM with selkies + Tailscale + KeePassXC

## Tests

One declarative check (build-scope):

- `keepassxc-binary` — `/usr/bin/keepassxc` executable exists

No deploy-scope tests: KeePassXC is a GUI app launched on-demand by the user inside the desktop session — there's no listening port or headless responder to probe.

## Relationship to `/ov:secrets`

This layer is the **GUI** for editing `.kdbx` databases. It is **distinct** from `/ov:secrets`, which is the ov CLI for a KeePass-backed credential store (separate `ov-secrets.kdbx` file, headless automation, kernel-keyring master-password cache).

- Author a `.kdbx` file in KeePassXC (GUI) → use it with `ov secrets init --kdbx path/to/file.kdbx` (CLI).
- The GUI and CLI share nothing except the KeePass file format.

## Alternative: bundled placement in `desktop-apps`

Before this layer existed, `keepassxc` shipped inside `/ov-layers:desktop-apps` alongside `btop`, `chromium`, `cockpit`, `transmission`, `vlc`, `zsh`. That's still the right choice when you want the whole bundle. Use this single-responsibility layer when you want KeePassXC without dragging in the rest.

## Related Skills

- `/ov-layers:desktop-apps` — bundle that also includes keepassxc (use when you want the full desktop-app set)
- `/ov-images:selkies-desktop-bootc` — primary consumer of this standalone layer
- `/ov-layers:selkies-desktop` — desktop composition this layer pairs with
- `/ov:secrets` — ov CLI credential store (shares `.kdbx` format only; different tool)
- `/ov:layer` — layer authoring reference
- `/ov:eval` — declarative testing reference

## When to Use This Skill

Use when the user asks about:

- Adding KeePassXC to a container or VM image without pulling in `desktop-apps`
- Why keepassxc lives in two places (standalone layer vs. desktop-apps bundle)
- Relationship between the KeePassXC GUI and `ov secrets`
