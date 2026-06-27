---
name: dbus-layer
description: |
  D-Bus session bus for inter-process communication inside containers.
  Use when working with D-Bus, desktop services, Wayland compositor dependencies, or charly check dbus commands.
---

# dbus -- D-Bus session bus service

## Candy Properties

| Property | Value |
|----------|-------|
| Dependencies | `supervisord` |
| Service | `dbus` (supervisord, priority 2) |
| Install files | `task:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `DBUS_SESSION_BUS_ADDRESS` | `unix:path=/tmp/dbus-session` |

## Packages

- `dbus-daemon` (RPM)

## Cross-distro coverage

RPM: `dbus-daemon` (Fedora) · PAC: `dbus` (Arch) · DEB: `dbus` (Debian/Ubuntu). Full parity — same DBUS_SESSION_BUS_ADDRESS env, same supervisord fragment, identical behavior downstream.

## Usage

```yaml
# charly.yml -- now included in all images with supervisord
my-image:
  candy:
    - dbus
```

## `charly check dbus` Integration

The `charly check dbus` command provides native Go D-Bus interaction with containers via `godbus/dbus/v5`. It operates in two modes:

- **Remote mode** (`charly check dbus notify myimage "title" "body"`): delegates to the container's `charly` binary via `engine exec container charly check dbus notify . "title" "body"`. Falls back to `gdbus call` if `charly` binary is not in the container.
- **Local mode** (`charly check dbus notify . "title" "body"`): connects directly to the local session bus. This is what runs inside the container when delegated from the host.

### Commands

```bash
charly check dbus notify <image> "title" "body"    # Send desktop notification
charly check dbus list <image>                      # List D-Bus services
charly check dbus call <image> <dest> <path> <method> [type:value...]  # Generic method call
charly check dbus introspect <image> <dest> <path>  # Service introspection
```

### Notification Delivery Chain

`charly cmd`/`charly tmux cmd` → `sendContainerNotification()` → `charly check dbus notify` → `org.freedesktop.Notifications.Notify` → swaync/mako → desktop popup

For notifications to work, the box needs:
1. **`dbus` candy** — D-Bus session bus (this candy)
2. **`swaync` candy** — notification daemon
3. **`charly` candy** — in-container charly binary for native D-Bus (falls back to `gdbus` from `glib2`)

### Error Messages

`charly check dbus` provides explanatory errors when D-Bus is unavailable:
- No session bus → suggests adding the `dbus` candy
- No notification daemon → suggests adding `swaync`
- No `charly` binary → falls back to `gdbus`, warns about adding `charly` candy

## `charly status` Probe

The `dbus` probe checks:
1. Whether `dbus-daemon` process is running
2. Which notification daemon is active (swaync, mako, dunst)

Shows as `dbus:ok (notify:swaync)` in `charly status` detail view.

## Used In Boxes

- Transitive dependency via `sway` in all desktop boxes
- Now explicitly added to all boxes with supervisord (openclaw, jupyter, ollama, immich, etc.)

## Related Candies

- `/charly-infrastructure:supervisord` -- process manager dependency
- `/charly-selkies:sway` -- primary consumer (Sway depends on dbus)
- `/charly-selkies:swaync` -- notification daemon (depends on dbus)
- `/charly-selkies:libnotify` -- `notify-send` CLI tool (depends on dbus)
- `/charly-tools:charly` -- in-container charly binary for native D-Bus

## Related Commands

- `/charly-check:dbus` — Native Go D-Bus commands (notify, list, call, introspect)

## When to Use This Skill

Use when the user asks about:

- D-Bus session bus configuration
- `DBUS_SESSION_BUS_ADDRESS` environment variable
- `charly check dbus` commands (notify, call, list, introspect)
- Desktop notifications in containers
- Inter-process communication in containers

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-check:check` — declarative testing (`check:` block, `charly check box`, `charly check live`)
