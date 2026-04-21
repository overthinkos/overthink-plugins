---
name: supervisord
description: |
  Supervisord process manager for running multiple services inside containers.
  Use when working with supervisord, container service management, multi-process containers,
  event listeners for crash-loop circuit breaking, or service priority ordering.
---

# supervisord -- process manager

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | **(none)** — system Python comes from the `supervisor` RPM's own dep |
| Install files | `layer.yml`, `supervisord.header.conf` (via `build.yml `init:` section` header_file) |
| Init role | Default init system for container images (set in `build.yml `init:` section`) |

## Packages

- `supervisor` (RPM) — process control system. The RPM brings
  `/usr/bin/python3` as its own dependency; supervisord's shebang is
  `/usr/bin/python3`. **No pixi-python / no conda-forge Python env is
  needed.**
- `supervisor` (pac) — same package name on Arch (`extra/` repo), same
  system-Python dependency.

### The vestigial `depends: python` removed in 2026-04

The layer used to declare `depends: python`, which pulled in the
`python` ov-layer → `pixi` ov-layer → a conda-forge Python env
(~500 MB). Nothing in supervisord's runtime touched that env — it's
pure system-Python all the way. The dep was dropped in 2026-04.
Every image in the catalog that uses supervisord (which is nearly all
of them) got an image-size drop of several hundred MB. See CLAUDE.md
"Key Rules" → *"Don't declare defensive deps"* for the general rule.

**Arch note:** the `pac: [supervisor]` section was added alongside the
intermediates-inheritance fix so that Arch-based images with
`build: [pac]` actually install supervisord. Previously only `rpm:` was
declared, so Arch images silently got a `/etc/supervisord.conf` with no
`supervisord` binary — the quadlet's `supervisord -n -c /etc/supervisord.conf`
exited 127 (command not found) at container start. Every Arch-rooted
auto-intermediate that composes `supervisord` was affected.

## How `ov` Generates Supervisord Configs

Layers declare processes via the unified **`services:`** schema in `layer.yml` (see `/ov:layer` "Service Declaration"). Each entry is rendered through supervisord's `service_schema.service_template` in `build.yml`, which produces a `[program:<name>]` INI fragment. `ov image generate` collects all rendered fragments across the layer chain and writes them into `/etc/supervisord.conf` inside the image, prefixed by the header from `templates/supervisord.header.conf` (referenced from `build.yml init:` section).

```yaml
# layers/chrome/layer.yml — unified schema (post-2026-04 migration)
services:
  - name: chrome
    exec: /home/user/.local/bin/chrome-wrapper --force-renderer-accessibility --no-first-run --start-maximized
    restart: always
    user: user
    env:
      # ...
    priority: 50                  # supervisord-specific; ordering hint
    stdout: file:/tmp/supervisor-chrome.log
    scope: system
    enable: true
```

The render template maps the abstract spec to supervisord INI:

| `services:` field | Supervisord output |
|---|---|
| `exec` | `command=` |
| `env` | `environment=K="V",K2="V2"` |
| `restart: always` | `autorestart=true` (helper `supervisordRestart`) |
| `restart: on-failure` | `autorestart=unexpected` |
| `restart: no` / unset | `autorestart=false` |
| `working_directory` | `directory=` |
| `user` | `user=` |
| `priority` | `priority=` (supervisord-specific) |
| `stop_timeout` | `stopwaitsecs=` |
| `stdout: file:/path` | `stdout_logfile=/path` (helper `supervisordLog`) |
| `stdout: journal` / unset | `stdout_logfile=/dev/stdout` |

**Supervisord does NOT support `use_packaged:`** — it doesn't consume systemd units. Entries with `use_packaged:` are skipped with a warning when rendering into a supervisord-init image. Use a custom entry (with explicit `exec:`) on supervisord-init images, or target a systemd init (bootc / host-deploy) where `use_packaged:` is honored.

The rendered INI lands at `/etc/supervisord.d/<layer>-<name>.conf` and is assembled into `/etc/supervisord.conf` at container-build time via the init system's fragment pipeline (`build.yml init.supervisord.fragment_template` + `stage_fragment_copy`).

At runtime, PID 1 is `supervisord`. The container's lifecycle is the supervisord process's lifecycle — when supervisord exits, the container exits (and the systemd quadlet's `Restart=always` rebuilds it).

### Legacy `service:` / `system_services:` — retired 2026-04

All 40 in-tree layers migrated from the legacy `service:` (raw supervisord INI block) and `system_services:` (list of unit names) fields to the unified `services:` list. The `service:` field is still parsed by the compiler's `compileServiceStepsLegacy` path for backwards compat with external (out-of-tree) layer sources. New in-tree layers should use `services:` only. See `/ov:layer` "Legacy fields" for the migration table.

## Service Priority & Ordering

Supervisord starts programs in **priority order** (ascending). Layers set priorities explicitly so that dependency chains come up in the right order:

| Priority | Typical services |
|----------|------------------|
| `1-9` | System-level: dbus, pipewire |
| `10` | Compositors: sway, labwc, mutter, kwin, niri |
| `15-20` | Desktop extras: waybar, swaync |
| `30-50` | Applications: chrome, openclaw, hermes |
| `100` | Auxiliary: cdp-proxy, chrome-devtools-mcp |
| `200+` | Last-start daemons: event listeners |

Programs with no `priority=` default to 999 and start last.

## autostart / autorestart / startretries

The knobs that shape crash-recovery behavior:

- `autostart=true` (default) — Start at container boot. Use `autostart=false` for services that must be hand-started (e.g., Chrome needs the compositor up first, so labwc's autostart does `supervisorctl start chrome`).
- `autorestart=true` (default) — Restart the program when it exits non-zero. Combined with `startretries`, this creates a crash-retry loop.
- `startretries=N` (default 3) — How many consecutive restarts supervisord will attempt before declaring the program `FATAL`. Once FATAL, supervisord stops trying.
- `startsecs=N` (default 1) — How long the program must stay up before supervisord considers the start "successful." Crashes within `startsecs` count against `startretries`; crashes after reset the counter.

**A program that can't stay up for `startsecs` seconds, `startretries` times in a row, is marked FATAL** and a `PROCESS_STATE_FATAL` event is emitted — which is what the event listener pattern below catches.

## Event Listeners

Supervisord supports a **plugin model** for reacting to process state changes: event listeners are separate programs that subscribe to events and can take arbitrary action. They run as supervisord programs themselves (using `[eventlistener:<name>]` instead of `[program:<name>]`) and communicate with supervisord over stdin/stdout using a simple protocol.

### Chrome crash-loop circuit breaker (canonical example)

The chrome layer (`/ov-layers:chrome`) ships `chrome-crash-listener`, which subscribes to `PROCESS_STATE_FATAL` and terminates supervisord (PID 1) with `SIGTERM` when Chrome enters the FATAL state.

```ini
[eventlistener:chrome-crash-listener]
command=/home/user/.local/bin/chrome-crash-listener
events=PROCESS_STATE_FATAL
autostart=true
autorestart=true
priority=200
user=user
```

**Why terminate PID 1?** Because Chrome's failure mode is typically a cgroup-level memory pressure problem — orphan shmem from prior crashes (memfd-backed IPC buffers, Skia texture pools) accumulates inside the container's cgroup and cannot be released by restarting Chrome alone (`chrome-restart` won't help). The only way to flush the cgroup is to tear down the container. By terminating supervisord, the container exits, the systemd quadlet's `Restart=always` rebuilds it, and the memory namespace is reset from scratch.

Pairs with the chrome layer's `security.memory_max`/`memory_high`/`memory_swap_max` cgroup caps — the cgroup triggers OOM-kill before the host runs out of memory, supervisord sees `autorestart=true` hit `startretries`, and the event listener escalates to container rebuild. See `/ov-layers:chrome` (Resource Caps & Circuit Breaker) for the full pattern and `/ov:layer` (Security Declaration → resource caps) for the merge semantics.

### Event listener protocol (for authoring your own)

An event listener is a shell script or Python program that:

1. Writes `READY\n` to stdout to tell supervisord it's ready for an event.
2. Reads a header line from stdin (format: `ver:3.0 server:supervisor serial:N pool:name poolserial:M eventname:NAME len:L`).
3. Reads `L` bytes of payload from stdin.
4. Takes action (log, email, kill PID 1, restart a service, etc.).
5. Writes `RESULT 2\nOK` to stdout.
6. Loops back to step 1.

Full protocol details: [supervisord event listener docs](http://supervisord.org/events.html). `supervisorctl avail` lists both regular programs and event listeners.

## Diagnostics

```bash
# Inside container
supervisorctl status           # All programs + states
supervisorctl avail            # All defined programs (including stopped)
supervisorctl pid              # Supervisord's own PID — best liveness probe
supervisorctl tail -f chrome stderr  # Live stderr log for one program
supervisorctl restart chrome   # Restart a single program (does not reset startretries counter)

# From host
ov service status <image>      # Wraps supervisorctl status via ov shell
ov service start/stop/restart <image> <name>   # Per-program control
ov logs <image>                # Container-level stdout/stderr (supervisord output)
```

### Declarative-testing liveness check

**`supervisorctl status` returns exit 3 when ANY program is non-RUNNING**
(FATAL, STOPPED, or EXITED). Layers with `autostart=false` programs
(e.g. `hermes` ships `hermes-whatsapp` as autostart=false) will always
fail a naive `command: supervisorctl status; exit_status: 0` test.

The robust liveness probe is `supervisorctl pid`, which asks
supervisord for its own PID and exits 0 iff the socket responds:

```yaml
- id: supervisorctl-responds
  scope: deploy
  command: supervisorctl pid
  exit_status: 0
  in_container: true
```

This is what the current supervisord layer ships in its `tests:` block.
See `/ov:test` Authoring Gotcha #4.

**Also note**: `pgrep` is NOT installed by default in minimal images
(needs `procps-ng`). The `process: <name>; running: true` test verb
silently fails when pgrep is absent. Prefer `service:` (which uses
supervisorctl internally) for program-liveness checks. See `/ov:test`
Authoring Gotcha #3.

`supervisorctl avail | grep -q '^<name>\b'` is the idiomatic way to check whether a program is defined (as opposed to whether it's currently running). This is what labwc's autostart uses to decide whether to hand off Chrome to supervisord or fall back to a direct launch — see `/ov-layers:labwc` (autostart Chrome-duplication race) for the canonical pattern.

## Header File

`templates/supervisord.header.conf` (at the repo root) defines global `[supervisord]`, `[unix_http_server]`, `[supervisorctl]`, and `[rpcinterface:supervisor]` sections. It's referenced from `build.yml `init:` section` as the init system's `header_file:` and prepended to every generated `supervisord.conf`. Common fields:

- `[unix_http_server] file=/tmp/supervisor.sock` — the socket `supervisorctl` uses
- `[supervisord] nodaemon=true logfile=/tmp/supervisord.log pidfile=/tmp/supervisord.pid`
- `[supervisord] user=user` — supervisord runs as the container user, not root

## Cross-distro coverage

RPM: `supervisor` (Fedora) · PAC: `supervisor` (Arch community) · DEB: `supervisor` (Debian/Ubuntu — package is named `supervisor`, not `supervisord`). Full parity; the process manager itself plus the header template is identical across distros. The `[supervisord] user=user` line in `templates/supervisord.header.conf` is a fallback only — when `user_policy: adopt` fires on an ubuntu base image, supervisord inherits the adopted `ubuntu` identity at runtime via the USER directive, so `user=user` in the header is harmless (supervisord's `user=` field is advisory — uid passthrough from the Dockerfile USER directive takes precedence).

## Usage

```yaml
# image.yml
my-image:
  layers:
    - supervisord
    - my-service  # layers with service: entries need supervisord
```

Adding a `service:` block to a layer automatically pulls in `supervisord` via `build.yml `init:` section`'s `depends_layer`. You rarely add `supervisord` to an image's `layers:` list manually.

## Used In Images

Transitive dependency for all images with managed services, including:
`openclaw`, `openclaw-sway-browser`, `openclaw-ollama-sway-browser`, `openclaw-ollama`, `jupyter`, `jupyter-ml`, `jupyter-ml-notebook`, `ollama`, `comfyui`, `immich`, `immich-ml`, `selkies-desktop`, `selkies-desktop-nvidia`, `selkies-desktop-bootc`, `hermes`, `openwebui`, `filebrowser`.

## Running supervisord under systemd (bootc mode)

On non-bootc images, supervisord is container PID 1 (`ENTRYPOINT=supervisord` emitted by the init system definition in `build.yml`). On **bootc** images, systemd is PID 1, which means supervisord needs a systemd unit wrapper — otherwise the whole desktop tier never starts. `/ov-layers:bootc-config` ships that wrapper as a systemd **user** unit at `/etc/systemd/user/supervisord.service`, `systemctl --global enable`d so tty1 autologin brings it up. See `/ov-layers:bootc-config` for the full mechanism (including the linger sentinel file).

### Two gotchas specific to running under a systemd user service

Both involve opening `/dev/stdout` or `/dev/fd/1`, which resolve to the journal pipe under a systemd user service — and `open()` on a pipe returns ENXIO.

1. **Main supervisord logfile.** The header template (`templates/supervisord.header.conf`) was changed from `logfile=/dev/stdout` to `logfile=/tmp/supervisord.log`. `/dev/stdout` works when supervisord is PID 1 in a container (fd 1 is real stdio), but fails with `OSError: [Errno 6] No such device or address: '/dev/stdout'` under a systemd user service where fd 1 is a pipe. Writing to a regular file works everywhere.
2. **Per-program `stdout_logfile=/dev/fd/1`.** Every layer's `service:` fragment redirects program stdout to `/dev/fd/1` so container logs (`podman logs`) show per-program output. Under a systemd user service this fails with `unknown error making dispatchers for <name>: ENXIO` for every program. The fix lives in the systemd user unit itself — set `StandardOutput=file:/tmp/supervisord-stdout.log` so supervisord's fd 1 backs a real file, not a pipe. Existing per-program `/dev/fd/1` lines then resolve correctly. See `/ov-layers:bootc-config` for the unit definition.

Container-mode logs are unaffected — supervisord is still PID 1 there.

## Related Layers

- `/ov-layers:python` -- Optional pixi-python env (NOT a dep of this layer as of 2026-04; supervisord uses system python3 from RPM)
- `/ov-layers:chrome` -- Canonical consumer of the crash-loop circuit breaker pattern (chrome-crash-listener, resource caps)
- `/ov-layers:labwc` -- Uses `supervisorctl avail` + `supervisorctl start chrome` to hand off to supervisord with a TOCTOU-safe sequence (autostart race fix in commit `febb9bd`)
- `/ov-layers:traefik` -- Reverse proxy (depends on supervisord)
- `/ov-layers:dbus` -- D-Bus session bus (depends on supervisord)
- `/ov-layers:bootc-config` -- ships the systemd-user supervisord autostart wrapper for bootc images
- `/ov-images:selkies-desktop-bootc` -- canonical worked example of supervisord running under systemd
- `/ov-layers:ollama`, `/ov-layers:openclaw`, `/ov-layers:postgresql`, `/ov-layers:redis`, `/ov-layers:sway`, `/ov-layers:mutter`, `/ov-layers:niri`, `/ov-layers:kwin` -- All ship `service:` blocks

## Related Commands

- `/ov:service` — Start/stop/restart/status for individual services inside the container
- `/ov:logs` — Container-level log access (shows supervisord-aggregated stdout/stderr)
- `/ov:status` — Container status including service probe results
- `/ov:generate` — Containerfile generation: where `service:` blocks get written to `/etc/supervisord.conf`
- `/ov:layer` — `service:` field authoring + `security:` resource caps that drive the circuit breaker

## When to Use This Skill

Use when the user asks about:

- Supervisord configuration or process management
- Running multiple services in a container
- Service startup order (priority ordering)
- autostart / autorestart / startretries behavior and crash-loop recovery
- Event listeners (especially the chrome-crash-listener circuit breaker pattern)
- `supervisorctl` commands (`status`, `avail`, `start`, `tail`, `restart`)
- The `ov service` commands (start/stop/restart/status)
- The `supervisor` RPM package or `/tmp/supervisor.sock`
