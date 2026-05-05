---
name: ov-full
description: |
  Full ov toolchain composition with CLI, virtualization, encrypted storage, and console access.
  Works identically on container/pod targets AND on host/local/bootc targets via the unified
  virtualization layer's mixed-`service:` schema. The previous ov-full-host sibling was
  deleted in the 2026-05 init-system-polymorphism cutover.
---

# ov-full -- Full ov toolchain (single layer for both pod and host targets)

## Layer Properties

| Property | Value |
|----------|-------|
| Layers (composition) | `ov`, `virtualization`, `gocryptfs`, `socat` |
| Install files | none (pure composition) |
| Target context | works for `kind: image` (container/pod), `kind: vm` (bootc/cloud_image), AND `kind: local` (host install) — the underlying `virtualization` layer handles init-system polymorphism via the mixed-entry `service:` pattern |

## What changed in the 2026-05 polymorphism cutover

Before: two sibling layers — `ov-full` (container/pod, used `virtualization` + `gvisor-tap-vsock` + `podman-machine`) and `ov-full-host` (host installs, used `virtualization-host` and dropped the container-only artifacts). They drifted because every change had to be applied twice.

After: ONE `ov-full` layer for both contexts. The container-only `gvisor-tap-vsock` + `podman-machine` packages were dropped entirely (legacy/unused per audit; nothing in the codebase invoked them). The unified `virtualization` layer carries BOTH a supervisord-rendered form (custom `exec:` for virtqemud/virtnetworkd) AND a systemd-rendered form (`use_packaged: virtqemud.socket` / `virtnetworkd.socket`) under the same `name:` — the init system at deploy time picks the matching form. See CLAUDE.md "Init-system polymorphism via mixed `service:` entries" for the rule and `/ov-foundation:virtualization` for the canonical worked example.

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

- `/ov-foundation:ov` -- ov CLI binary (included)
- `/ov-foundation:virtualization` -- QEMU/KVM/libvirt stack with mixed-entry `service:` for both supervisord and systemd (included; canonical worked example of the polymorphism pattern)
- `/ov-foundation:gocryptfs` -- encrypted filesystem for `ov config` encrypted volumes (included)
- `/ov-foundation:socat` -- socket relay for console access and port_relay (included)
- `/ov-foundation:bootc-base` -- often paired for OS images

## Used In Images

- `/ov-coder:arch-ov`
- `/ov-foundation:fedora-ov`
- `/ov-foundation:githubrunner`
- `/ov-foundation:aurora` (disabled)
- Operator host install via the `ov-cachyos` kind:local template (post-2026-05)

## When to Use This Skill

Use when the user asks about:

- Running ov inside containers, VMs, or directly on a host
- VM management toolchain composition
- Encrypted storage support
- The "I need ov tools available" composition
- Why the previous `ov-full-host` no longer exists

## Related

- `/ov-build:layer` — layer authoring; "Service Declaration" + "Anti-pattern: `<name>-host` / `<name>-pod` sibling layers" subsections
- `/ov-foundation:supervisord` — init system documentation for container-side rendering
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov eval live`)
- CLAUDE.md "Init-system polymorphism via mixed `service:` entries" Key Rule
