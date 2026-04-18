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
| Dependencies | `python` |
| Install files | `layer.yml`, `supervisord.header.conf` (via `init.yml` header_file) |
| Init role | Default init system for container images (set in `init.yml`) |

## Packages

- `supervisor` (RPM) -- process control system (Python-based)

## How `ov` Generates Supervisord Configs

Layers declare processes via `service:` in `layer.yml` — a raw supervisord `[program:<name>]` fragment. `ov image generate` collects all `service:` fragments across the layer chain and writes them into `/etc/supervisord.conf` inside the image, prefixed by the header from `templates/supervisord.header.conf` (referenced from `init.yml`).

```yaml
# layers/chrome/layer.yml
service: |
  [program:chrome]
  command=/home/user/.local/bin/chrome-wrapper --force-renderer-accessibility --no-first-run --start-maximized
  autostart=false
  autorestart=true
  startretries=3
  startsecs=5
  user=user
  environment=...
  stdout_logfile=/tmp/supervisor-chrome.log
  stderr_logfile_maxbytes=0
```

At runtime, PID 1 is `supervisord`. The container's lifecycle is the supervisord process's lifecycle — when supervisord exits, the container exits (and the systemd quadlet's `Restart=always` rebuilds it).

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
supervisorctl tail -f chrome stderr  # Live stderr log for one program
supervisorctl restart chrome   # Restart a single program (does not reset startretries counter)

# From host
ov service status <image>      # Wraps supervisorctl status via ov shell
ov service start/stop/restart <image> <name>   # Per-program control
ov logs <image>                # Container-level stdout/stderr (supervisord output)
```

`supervisorctl avail | grep -q '^<name>\b'` is the idiomatic way to check whether a program is defined (as opposed to whether it's currently running). This is what labwc's autostart uses to decide whether to hand off Chrome to supervisord or fall back to a direct launch — see `/ov-layers:labwc` (autostart Chrome-duplication race) for the canonical pattern.

## Header File

`templates/supervisord.header.conf` (at the repo root) defines global `[supervisord]`, `[unix_http_server]`, `[supervisorctl]`, and `[rpcinterface:supervisor]` sections. It's referenced from `init.yml` as the init system's `header_file:` and prepended to every generated `supervisord.conf`. Common fields:

- `[unix_http_server] file=/tmp/supervisor.sock` — the socket `supervisorctl` uses
- `[supervisord] nodaemon=true logfile=/tmp/supervisord.log pidfile=/tmp/supervisord.pid`
- `[supervisord] user=user` — supervisord runs as the container user, not root

## Usage

```yaml
# images.yml
my-image:
  layers:
    - supervisord
    - my-service  # layers with service: entries need supervisord
```

Adding a `service:` block to a layer automatically pulls in `supervisord` via `init.yml`'s `depends_layer`. You rarely add `supervisord` to an image's `layers:` list manually.

## Used In Images

Transitive dependency for all images with managed services, including:
`openclaw`, `openclaw-sway-browser`, `openclaw-ollama-sway-browser`, `openclaw-ollama`, `jupyter`, `jupyter-ml`, `jupyter-ml-notebook`, `ollama`, `comfyui`, `immich`, `immich-ml`, `selkies-desktop`, `selkies-desktop-nvidia`, `hermes`, `openwebui`, `filebrowser`.

## Related Layers

- `/ov-layers:python` -- Python runtime dependency (supervisord is Python-based)
- `/ov-layers:chrome` -- Canonical consumer of the crash-loop circuit breaker pattern (chrome-crash-listener, resource caps)
- `/ov-layers:labwc` -- Uses `supervisorctl avail` + `supervisorctl start chrome` to hand off to supervisord with a TOCTOU-safe sequence (autostart race fix in commit `febb9bd`)
- `/ov-layers:traefik` -- Reverse proxy (depends on supervisord)
- `/ov-layers:dbus` -- D-Bus session bus (depends on supervisord)
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
