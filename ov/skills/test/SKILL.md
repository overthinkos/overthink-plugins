---
name: test
description: |
  MUST be invoked before any work involving: `ov test`, `ov image test`, the
  `tests:` / `deploy_tests:` fields in layer.yml / image.yml / deploy.yml, the
  `org.overthinkos.tests` OCI label, or any declarative check authoring.
  Covers verb catalog (file/port/command/http/package/service/process/dns/
  user/group/interface/kernel-param/mount/addr/matching), runtime variable
  resolution (`${HOST_PORT:N}`, `${VOLUME_PATH:name}`, `${CONTAINER_IP}`,
  `${ENV_*}`), deploy.yml overlay rules, authoring gotchas learned the hard
  way (package renames, absent binaries, host vs container network routing),
  and both build-time / deploy-time CLI wrappers.
---

# Test - Declarative Full-Stack Testing

## Overview

`ov` ships a goss-inspired declarative testing framework built into the
CLI. Tests are authored inline under `tests:` (or `deploy_tests:`) in
`layer.yml`, `image.yml`, or `deploy.yml`. They are **embedded as a
three-section OCI label** (`org.overthinkos.tests` → `{layer, image, deploy}`)
so any pulled image is self-testable without its source repo. A local
`deploy.yml` overlay can add checks or override baked ones by `id:`.

The runner resolves **deploy-time variables** (actual host port mappings,
volume backings, env vars, container IP, DNS) at execution time, so a
check written once works unchanged when `deploy.yml` remaps ports.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Run tests against running service | `ov test <image> [-i instance]` | Full three-section run + deploy overlay |
| Run tests against disposable container | `ov image test <image>` | Layer + image sections (deploy needs `--include-deploy`) |
| Validate authored tests at config time | `ov image validate` | Schema, scope/variable consistency, id uniqueness |
| Inspect effective spec | `ov image inspect <image>` | JSON includes merged tests structure |
| Filter by verb | `ov test <image> --filter file --filter port` | Repeatable |
| Filter by section | `ov test <image> --section deploy` | One of: layer / image / deploy |
| Output format | `ov test <image> --format json\|tap\|text` | Default text |

## Authoring: the `tests:` list

Every test is a **list entry with exactly one verb discriminator** plus
shared modifiers and verb-specific attributes. This mirrors the `tasks:`
pattern in `layer.yml`.

### Gold-standard pattern (redis layer)

```yaml
# layers/redis/layer.yml
tests:
  # Build-scope — run inside the built image via `podman run --rm`.
  - id: redis-binary
    file: /usr/bin/redis-server
    exists: true
  - id: redis-cli-binary
    file: /usr/bin/redis-cli
    exists: true
  - id: redis-package
    package: valkey-compat-redis     # see "Authoring Gotchas" on Fedora 43 renames
    installed: true

  # Deploy-scope — HOST_PORT:N resolves to the effective port mapping, so
  # the same test works whether the deploy publishes 6379:6379 or 16379:6379.
  - id: redis-responds
    scope: deploy
    command: redis-cli -h 127.0.0.1 -p ${HOST_PORT:6379} ping
    stdout: PONG
    in_container: false    # run redis-cli from the host
  - id: redis-port-open
    scope: deploy
    addr: 127.0.0.1:${HOST_PORT:6379}
    reachable: true
```

The six entries cover: binary existence, package identity, a functional
live probe, and raw TCP reachability. Adopt this shape for any primary
service layer.

### Verb catalog (goss-parity feature set)

| Verb | Typical attributes | Notes |
|------|---------------------|-------|
| `file` | `exists`, `mode`, `owner`, `group_of`, `filetype`, `contains`, `sha256` | `group_of` (not `group`) avoids the verb collision |
| `package` | `installed`, `versions` | Tries rpm / dpkg / pacman — exact match on installed name (see gotchas) |
| `service` | `running`, `enabled` | Supervisord first, then systemd |
| `port` | `listening`, `ip`, `reachable` | **`listening` needs `ss`/`netstat` inside the container** — absent from minimal images |
| `process` | `running` | **Needs `pgrep`** — absent from minimal images |
| `command` | `exit_status`, `stdout`, `stderr`, `in_container` | Default runs via `podman exec`; set `in_container: false` to run from host |
| `http` | `status` (int, single value), `body`, `headers`, `method`, `request_body`, `allow_insecure`, `no_follow_redirects`, `ca_file`, `timeout` | Host-side under `ov test`; curl-in-container under `ov image test` |
| `dns` | `resolvable`, `addrs`, `server` | Host resolver for `ov test`; `getent hosts` for `ov image test` |
| `user` | `uid`, `gid`, `home`, `shell` | `getent passwd` |
| `group` | `gid`, `groups` | `getent group` |
| `interface` | `mtu`, `addrs` | `ip -o addr show` |
| `kernel-param` | `value` (scalar or matcher) | `sysctl -n` |
| `mount` | `mount_source`, `filesystem`, `opts` | `findmnt` |
| `addr` | `reachable`, `timeout` | Pure Go-native `net.DialTimeout` for `ov test`; `nc` for `ov image test` |
| `matching` | `contains` | Pure in-process value matching — no target probe |

### Shared modifiers

| Field | Purpose |
|-------|---------|
| `id` | Optional stable identifier. Enables `deploy.yml` to override by `id`. Unique per section per image. |
| `description` | Human-readable label for reports. |
| `skip: true` | Always skip this check (reported but doesn't fail the run). |
| `timeout: "5s"` | Per-check timeout (http, addr). |
| `scope: build\|deploy` | Default `build` at layer/image level, `deploy` at deploy level. |

### Matcher forms

Any `MatcherList` attribute (`stdout`, `stderr`, `body`, `headers`,
`contains`, `opts`, `value`) accepts three shapes — picked by yaml.v3
at parse time:

```yaml
stdout: PONG                         # scalar  → [{equals: PONG}]
stdout: [PONG, READY]                # list    → [{equals: PONG}, {equals: READY}]
stdout:
  - equals: PONG                     # operator map
  - contains: ["ready", "ok"]
  - matches: "^[A-Z]+$"
  - not_contains: "error"
```

Supported operators: `equals`, `not_equals`, `contains`, `not_contains`,
`matches`, `not_matches`, plus `lt`/`le`/`gt`/`ge` (numeric, added as
verbs need them).

**`status:` is NOT a MatcherList** — it's a plain `int` on the `http`
verb. One code per test. See Authoring Gotcha #2.

## Authoring Gotchas (learned the hard way)

These are the non-obvious issues that surface only when you run the tests
against real containers. Skim before writing new checks.

### 1. Host-side deploy tests: `127.0.0.1:${HOST_PORT:N}`, NOT `${CONTAINER_IP}:${HOST_PORT:N}`

In rootless podman, the container's pod-network IP (e.g. `10.89.0.x`) is
not routable from the host. `${HOST_PORT:N}` resolves to a port on the
host's loopback that forwards into the container. Combining them is
nonsense:

```yaml
# ❌ WRONG — 10.89.0.x:HOST_PORT is unreachable from either side.
addr: ${CONTAINER_IP}:${HOST_PORT:8080}

# ✓ RIGHT — host-side access via the forwarded port.
addr: 127.0.0.1:${HOST_PORT:8080}

# ✓ ALSO RIGHT — in-container, using the container port directly.
command: curl -fsS http://127.0.0.1:8080/health
in_container: true
```

Use `${CONTAINER_IP}` only from inside another container in the same pod.

### 2. `http` status is a single int — no lists

```yaml
# ❌ WRONG — yaml unmarshals a list to int fails with "cannot unmarshal !!seq"
status: [200, 301, 302]

# ✓ RIGHT — pick the most-likely code. Or author multiple tests.
status: 200
```

For endpoints that return 400/401 when probed without auth (MCP servers,
API endpoints), `status: 400` is a legitimate "endpoint exists and refuses
empty input" check.

### 3. `pgrep`, `ss`, `nc` are NOT in minimal images

The `process:`, `port: listening`, and `nc`-based command tests silently
fail on images that skip `procps-ng` / `iproute` / `ncat`. Portable
alternatives:

| Intent | Portable alternative |
|--------|----------------------|
| "supervisord is running" | `command: supervisorctl pid; exit_status: 0` (in_container) |
| "some program under supervisord" | `service: <name>; running: true` |
| "port N listening in-container" | `command: curl -fsS http://127.0.0.1:N/<route>; in_container: true` |
| "port reachable from host" | `addr: 127.0.0.1:${HOST_PORT:N}; reachable: true` (Go-native dial) |

### 4. `supervisorctl status` exits 3 when ANY program is non-RUNNING

Layers like `hermes` declare `autostart=false` programs (e.g.
`hermes-whatsapp`), which leaves them STOPPED. `supervisorctl status`
returns exit 3 — fails `exit_status: 0` checks.

```yaml
# ❌ BRITTLE — fails if any program is autostart=false.
command: supervisorctl status
exit_status: 0

# ✓ ROBUST — `pid` asks for supervisord's own PID, 0 iff the socket
# responds.
command: supervisorctl pid
exit_status: 0
in_container: true
```

### 5. `ov version` writes to stderr, not stdout

Common gotcha — also affects OpenSSH's `ssh -V` and others. Use
`stderr:` matcher:

```yaml
- id: ov-version
  command: /usr/local/bin/ov version
  exit_status: 0
  stderr:
    - matches: "[0-9]{4}\\.[0-9]+"
```

### 6. Drop `$` anchor in `matches:` regexes on command output

Command stdout typically ends with `\n`. Go's `regexp` treats `$` as
end-of-input (not end-of-line by default), so `\.ipynb$` won't match
`path.ipynb\n`. Drop the anchor:

```yaml
# ❌ WRONG — fails because trailing newline
stdout:
  - matches: "\\.ipynb$"

# ✓ RIGHT — substring match is usually what you want
stdout:
  - matches: "\\.ipynb"
```

### 7. `${VAR:-default}` is NOT supported by ov's resolver

ov's variable resolver accepts `${IDENT}` only — no bash-style defaults,
no parameter expansion. An unresolved variable is reported as a
`unresolved variables: <name>` SKIP.

```yaml
# ❌ Parsed as a single var name; reported as unresolved if not set
command: psql -U ${POSTGRES_USER:-postgres}

# ✓ Plain env-var reference; resolves from the container's environment
# at deploy scope. For a true default, set it in the layer's `env:`.
command: psql -U ${POSTGRES_USER}
```

### 8. Fedora 43 package renames — query the installed name, not the dnf request

The `package:` verb does exact string equality against the rpm
database, not the dnf install request. When dnf resolves a virtual
provides, the test must name the actual installed package.

| dnf request | Installed name on Fedora 43 |
|-------------|-----------------------------|
| `redis` | `valkey-compat-redis` |
| `liberation-fonts` | `liberation-sans-fonts` (+ `-serif-fonts`, `-mono-fonts`) |

Always verify with `podman run --rm <image> rpm -qf /path/to/binary`.

### 9. Volume paths don't exist at build-scope

`${HOME}/workspace`, `${VOLUME_CONTAINER_PATH:name}`, and similar paths
are provisioned by `ov config` at deploy time. A build-scope test on
them fails in `ov image test` because the mount isn't attached.

```yaml
# ❌ Fails under `ov image test` — workspace isn't mounted
- id: workspace-dir
  file: ${HOME}/workspace
  filetype: directory

# ✓ Scope to deploy so it runs only against live containers
- id: workspace-dir
  scope: deploy
  file: ${HOME}/workspace
  filetype: directory
```

### 10. Disposable tests run as the image's default USER — not root

`ov image test` runs `podman run --rm <image>` with the image's USER
directive (typically uid 1000). Root-only files (e.g. `/etc/sudoers.d/*`
is root:root 0750) report as "missing" because the user can't traverse
the parent directory.

```yaml
# ❌ Reports "exists=false" even though the file IS there (0750 dir)
- id: sudoers-ov-user
  file: /etc/sudoers.d/ov-user
  exists: true

# ✓ Verify the semantic (can I sudo?) instead of the file bit
- id: sudoers-ov-user
  command: sudo -n -l
  exit_status: 0
  stdout:
    - contains: "NOPASSWD"
```

## Three levels, three sections

| Section | Authored in | When it runs |
|---------|-------------|--------------|
| `layer` | `tests:` in `layers/<name>/layer.yml` (scope:"build") | `ov image test` + `ov test` |
| `image` | `tests:` in `image.yml` per image (scope:"build") | `ov image test` + `ov test` |
| `deploy` | `tests:` with `scope: deploy`, or `deploy_tests:` in `image.yml`, or local `deploy.yml` `tests:` | `ov test` always; `ov image test --include-deploy` |

The build label `org.overthinkos.tests` contains all three sections with
`origin:` annotations (`layer:<name>`, `image:<name>`, `deploy-default`,
`deploy-local`). `CollectTests` walks the base-image chain with a
visited-image guard: cycles are reported by `validateImageDAG` at validate
time, but the collector itself terminates cleanly even if called on a
pathological config.

## Verb routing (which executor runs each check)

Every verb dispatches through one of two executors depending on the run
mode — this is what makes the same `tests:` list work both against a
disposable container (`ov image test`) and a running service (`ov test`):

| Verb + attributes | Under `ov test` (running service) | Under `ov image test` (disposable) |
|-------------------|-----------------------------------|------------------------------------|
| file, package, service, process, user, group, interface, kernel-param, mount | `ContainerExecutor` (`podman exec`) | `ImageExecutor` (`podman run --rm`) |
| port with `listening` | `ContainerExecutor` (`ss`/`netstat` inside) | `ImageExecutor` |
| port with `reachable` | Host-side `net.DialTimeout` | Skipped (no host port binding on a disposable run) |
| command (`in_container: true`, default) | `ContainerExecutor` | `ImageExecutor` |
| command (`in_container: false`) | Host-side `exec.Command` | Skipped |
| http, dns, addr | Host-side (from the `ov` process) | In-container `curl` / `getent hosts` / `nc` |
| matching | In-process matcher eval | Same |

The routing table lives in `ov/testrun.go` (`runOne` switch) and
`ov/testrun_verbs.go`. When a check is unroutable (e.g. `port:
reachable` under `ov image test`), the runner reports it as **skipped**
with a reason rather than failing the run.

## Runtime variables

`ov test` resolves these via `podman inspect` on the running container
before any check executes. `${NAME:arg}` is parameterized form.

| Variable | Source | Scope |
|----------|--------|-------|
| `${USER}`, `${HOME}`, `${UID}`, `${GID}` | Image metadata (OCI labels) | build + deploy |
| `${IMAGE}` | Image metadata | build + deploy |
| `${DNS}`, `${ACME_EMAIL}` | Image metadata + deploy.yml overlay | build + deploy |
| `${INSTANCE}` | `--instance` flag / deploy.yml key | deploy |
| `${CONTAINER_NAME}`, `${CONTAINER_IP}` | `podman inspect` | deploy |
| `${HOST_PORT:N}` | Port mapping for container port N | deploy |
| `${VOLUME_PATH:name}` | Host path backing the named volume (bind source, encrypted mount, or `_data` dir) | deploy |
| `${VOLUME_CONTAINER_PATH:name}` | In-container mount path for a volume | deploy |
| `${ENV_NAME}` | Effective env var value on the running container | deploy |

Build-scope checks may **not** reference deploy-scope variables — the
validator flags this at `ov image validate` time.

**No bash-style defaults**: `${VAR:-fallback}` is unsupported (see
Authoring Gotcha #7). Plain `${VAR}` only.

## Deploy.yml overlay rules

A local `deploy.yml` can contribute its own `tests:` list per image.
Merge rules applied by `ov test`:

1. Local entries with an `id:` matching a baked entry replace that entry.
2. Entries without a matching `id:` are appended.
3. To disable a baked check, reference it with `id:` and `skip: true`.

```yaml
# ~/.config/ov/deploy.yml
images:
  redis-ml:
    tests:
      - id: redis-responds                  # overrides image's baked check
        command: redis-cli -h 127.0.0.1 -p 16379 ping
        stdout: PONG
        in_container: false
      - id: external-tunnel                 # appended (no baked match)
        http: https://redis.tailnet.ts.net/health
        status: 200
      - id: old-probe                       # disable a legacy baked check
        skip: true
```

## Typical workflow

```bash
# 1. Author tests in layers/<name>/layer.yml.
# 2. Validate schema + references.
ov image validate

# 3. Build the image (tests are auto-embedded as LabelTests).
#    LABELs are emitted LAST in the final stage — a test edit rebuilds
#    in ~2 sec (cache hits every upstream RUN/COPY).
ov image build redis-ml

# 4. Run against a disposable container (build-scope checks only).
ov image test redis-ml

# 5. Start the service and test end-to-end.
ov start redis-ml
ov test redis-ml

# 6. Override a baked deploy check via local deploy.yml, re-test.
$EDITOR ~/.config/ov/deploy.yml    # add tests: entry with matching id
ov test redis-ml
```

## Coverage snapshot (7 currently-tested images)

Reference numbers from the last end-to-end session:

| Image | Tests | Pass | Fail | Skipped |
|-------|-------|------|------|---------|
| `filebrowser` | 24 | 24 | 0 | 0 |
| `jupyter` | 29 | 29 | 0 | 0 |
| `openwebui` | 24 | 24 | 0 | 0 |
| `hermes` | 50 | 50 | 0 | 0 |
| `immich-ml` | 63 | 61 | 0 | 2 (redis port internal) |
| `selkies-desktop` | 91 | 91 | 0 | 0 |
| `sway-browser-vnc` | 85 | 84 | 0 | 1 (port 9224 not exposed here) |

The "skipped" entries are intentional — they reference ports that
aren't mapped in the containing image's `ports:` block. Skipping them
is correct behavior; they'd pass in an image that does expose those
ports.

## Related skills

- `/ov:layer` — layer authoring; `tests:` field is part of every `layer.yml`.
- `/ov:image` — image-level `tests:` and `deploy_tests:` at composition time.
- `/ov:deploy` — local `deploy.yml` overlay rules and the `tests:` merge.
- `/ov:validate` — static schema + cross-scope variable checks; the first
  gate before `ov image build`.
- `/ov:build` — how tests are embedded into the OCI label at build time;
  LABELs-at-end cache behavior.
- `/ov:inspect` — view the merged 3-section tests structure as JSON.
- `/ov-dev:go` — implementation map: `testspec.go`, `testvars.go`,
  `testrun.go`, `testrun_verbs.go`, `testcollect.go`, `test_cmd.go`,
  `validate_tests.go`, plus the `LabelTests` constant in `labels.go`.
- `/ov-dev:generate` — how `LabelTests` is written into the Containerfile
  via `writeJSONLabel`, and why the LABEL block lives at the end of the
  final stage.

## When to Use This Skill

**MUST be invoked** before authoring, running, or debugging declarative
tests at any level. If the user mentions `ov test`, `ov image test`,
`tests:`, `deploy_tests:`, `LabelTests`, or any check verb by name
(file/port/http/command/package/service/...), invoke this skill first.
