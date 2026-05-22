---
name: ov-full
description: |
  Full ov toolchain composition with CLI, virtualization, encrypted storage, and console access.
  Works identically on container/pod targets AND on host/local/bootc targets via the unified
  virtualization layer's mixed-`service:` schema — one layer for every target, no `-host` sibling.
---

# ov-full -- Full ov toolchain (single layer for both pod and host targets)

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `ov`, `virtualization`, `gocryptfs`, `socat` |
| Install files | none (pure composition) |
| Target context | works for `kind: image` (container/pod), `kind: vm` (bootc/cloud_image), AND `kind: local` (host install) — the underlying `virtualization` layer handles init-system polymorphism via the mixed-entry `service:` pattern |

## How one layer serves both pod and host targets

ONE `ov-full` layer covers both contexts. The unified `virtualization` layer carries BOTH a supervisord-rendered form (custom `exec:` for virtqemud/virtnetworkd) AND a systemd-rendered form (`use_packaged: virtqemud.socket` / `virtnetworkd.socket`) under the same `name:` — the init system at deploy time picks the matching form. See CLAUDE.md "Init-system polymorphism via mixed `service:` entries" for the rule and `/ov-infrastructure:virtualization` for the canonical worked example.

## Usage

```yaml
# overthink.yml
image:
  my-vm-host:
    layers:
      - ov-full           # works on pod images

local:
  my-host-profile:
    layers:
      - ov-full           # works on host installs (target: local)
```

The same layer reference works for both shapes; no `-host` variant is needed or available.

## Related Layers

- `/ov-tools:ov` -- ov CLI binary (included)
- `/ov-infrastructure:virtualization` -- QEMU/KVM/libvirt stack with mixed-entry `service:` for both supervisord and systemd (included; canonical worked example of the polymorphism pattern)
- `/ov-infrastructure:gocryptfs` -- encrypted filesystem for `ov config` encrypted volumes (included)
- `/ov-infrastructure:socat` -- socket relay for console access and port_relay (included)
- `/ov-distros:bootc-base` -- often paired for OS images

## Used In Images

- `/ov-coder:arch-ov`
- `/ov-distros:fedora-ov`
- `/ov-distros:githubrunner`
- `/ov-distros:aurora` (disabled)
- Operator host install via the `ov-cachyos` kind:local template

## When to Use This Skill

Use when the user asks about:

- Running ov inside containers, VMs, or directly on a host
- VM management toolchain composition
- Encrypted storage support
- The "I need ov tools available" composition
- Why one `ov-full` layer serves pod, VM, and host targets (no `-host` variant)

## Related

- `/ov-image:layer` — layer authoring; "Service Declaration" + "Anti-pattern: `<name>-host` / `<name>-pod` sibling layers" subsections
- `/ov-infrastructure:supervisord` — init system documentation for container-side rendering
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
- CLAUDE.md "Init-system polymorphism via mixed `service:` entries" Key Rule
