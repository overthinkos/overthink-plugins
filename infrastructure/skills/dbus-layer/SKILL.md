---
name: dbus-layer
description: |
  D-Bus session bus for inter-process communication inside containers.
  Use when working with D-Bus, desktop services, Wayland compositor dependencies, or charly eval dbus commands.
---

# dbus -- D-Bus session bus service

## Layer Properties

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
# box.yml -- now included in all images with supervisord
my-image:
  layers:
    - dbus
```

## `charly eval dbus` Integration

The `charly eval dbus` command provides native Go D-Bus interaction with containers via `godbus/dbus/v5`. It operates in two modes:

- **Remote mode** (`charly eval dbus notify myimage "title" "body"`): delegates to the container's `ov` binary via `engine exec container charly eval dbus notify . "title" "body"`. Falls back to `gdbus call` if `ov` binary is not in the container.
- **Local mode** (`charly eval dbus notify . "title" "body"`): connects directly to the local session bus. This is what runs inside the container when delegated from the host.

### Commands

```bash
charly eval dbus notify <image> "title" "body"    # Send desktop notification
charly eval dbus list <image>                      # List D-Bus services
charly eval dbus call <image> <dest> <path> <method> [type:value...]  # Generic method call
charly eval dbus introspect <image> <dest> <path>  # Service introspection
```

### Notification Delivery Chain

`charly cmd`/`charly tmux cmd`/`charly eval record cmd` → `sendContainerNotification()` → `charly eval dbus notify` → `org.freedesktop.Notifications.Notify` → swaync/mako → desktop popup

For notifications to work, the image needs:
1. **`dbus` layer** — D-Bus session bus (this layer)
2. **`swaync` layer** — notification daemon
3. **`ov` layer** — in-container charly binary for native D-Bus (falls back to `gdbus` from `glib2`)

### Error Messages

`charly eval dbus` provides explanatory errors when D-Bus is unavailable:
- No session bus → suggests adding the `dbus` layer
- No notification daemon → suggests adding `swaync`
- No `ov` binary → falls back to `gdbus`, warns about adding `ov` layer

## `charly status` Probe

The `dbus` probe checks:
1. Whether `dbus-daemon` process is running
2. Which notification daemon is active (swaync, mako, dunst)

Shows as `dbus:ok (notify:swaync)` in `charly status` detail view.

## Used In Images

- Transitive dependency via `sway` in all desktop images
- Now explicitly added to all images with supervisord (openclaw, jupyter, ollama, immich, etc.)

## Related Layers

- `/charly-infrastructure:supervisord` -- process manager dependency
- `/charly-selkies:sway` -- primary consumer (Sway depends on dbus)
- `/charly-selkies:swaync` -- notification daemon (depends on dbus)
- `/charly-selkies:libnotify` -- `notify-send` CLI tool (depends on dbus)
- `/charly-tools:charly` -- in-container charly binary for native D-Bus

## Related Commands

- `/charly-eval:dbus` — Native Go D-Bus commands (notify, list, call, introspect)

## When to Use This Skill

Use when the user asks about:

- D-Bus session bus configuration
- `DBUS_SESSION_BUS_ADDRESS` environment variable
- `charly eval dbus` commands (notify, call, list, introspect)
- Desktop notifications in containers
- Inter-process communication in containers

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
