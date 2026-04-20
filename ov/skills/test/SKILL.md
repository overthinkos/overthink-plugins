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
| Run tests against *live* container (auto-fallback) | `ov image test <image> --include-deploy` | Prefers the running `ov-<image>` container; falls back to disposable if none (see "Live vs disposable executor selection" below) |
| Validate authored tests at config time | `ov image validate` | Schema, scope/variable consistency, id uniqueness |
| Inspect effective spec | `ov image inspect <image>` | JSON includes merged tests structure |
| Filter by verb | `ov test <image> --filter file --filter port` | Repeatable |
| Filter by section | `ov test <image> --section deploy` | One of: layer / image / deploy |
| Output format | `ov test <image> --format json\|tap\|text` | Default text |

## Live vs disposable executor selection (`ov image test`, 2026-04)

`ov image test` picks its executor based on `--include-deploy`:

- **Without `--include-deploy`** — always spawns a disposable
  container via `podman run --rm <image-ref>` (`ImageExecutor`). Only
  layer + image sections run. This is the correct choice for pure
  build-scope invariants ("did this binary land at this path?") where
  a running service would only add noise.
- **With `--include-deploy`** — probes for `ov-<meta.Image>`
  **running** container. If alive: uses `ContainerExecutor`
  (`podman exec`) + `ResolveTestVarsRuntime` so deploy tests see real
  supervisord state, real `HOST_PORT:<N>` mappings (including
  host-networked containers via `HostConfig.NetworkMode` detection,
  new in 2026-04), and real in-container env. If no running
  container: falls back to `ImageExecutor` (same behaviour as the
  pre-2026-04 default — most deploy tests will skip or fail, which is
  the honest outcome).

The banner after the test run reports which executor was used:

```
Image: ghcr.io/overthinkos/fedora-coder:latest (live container: ov-fedora-coder)
# ← live

Image: ghcr.io/overthinkos/fedora-coder:latest
# ← disposable
```

The `meta.Image` short-name (from the `org.overthinkos.image` OCI
label) is used for the container-name lookup — full image refs like
`ghcr.io/overthinkos/fedora-coder:latest` are correctly mapped to
`ov-fedora-coder`. Implementation: `ov/test_cmd.go` `ImageTestCmd.Run()`.

## Subcommands: live-container verbs

`ov test` is both the declarative test runner AND the grouping point for the
interactive verbs that drive a running service. Kong's `default:"withargs"`
tag means `ov test <image>` still dispatches to the runner — only the
explicit subcommand names below take over when matched.

| Subcommand | Skill | Purpose |
|-----------|-------|---------|
| `ov test cdp …` | `/ov:cdp` | Chrome DevTools Protocol — open/list/close tabs, click, type, eval, screenshot, axtree |
| `ov test wl …` | `/ov:wl` | Wayland desktop input/windows/clipboard + sway IPC |
| `ov test dbus …` | `/ov:dbus` | D-Bus calls, introspection, desktop notifications |
| `ov test vnc …` | `/ov:vnc` | VNC framebuffer screenshot + click/key/type/passwd |
| `ov test mcp …` | (this skill) | MCP client — ping/list-tools/list-resources/list-prompts/call/read/servers against any `mcp_provides` endpoint. Speaks `github.com/modelcontextprotocol/go-sdk` (Streamable HTTP by default, SSE when `transport: sse`). |

These four verbs were previously top-level commands (`ov cdp`, `ov wl`,
`ov dbus`, `ov vnc`). They moved under `ov test` because every one of them
is a "probe or drive a running service" operation — the same surface the
declarative test runner composes when it executes checks. The old top-level
forms were removed (no deprecation shim).

**Reserved image names:** because subcommand names take priority when
matched, an image literally named `cdp`, `wl`, `dbus`, `vnc`, or `mcp`
cannot be run via `ov test <name>` — use the explicit `ov test run <name>`
form or rename the image. No such images currently exist in `image.yml`.

**Gotcha — stale container-baked `ov` binary:** `ov test dbus notify` and
`ov test dbus call` delegate to the container's own `ov` binary (see
`ov/notify.go:20`, `ov/dbus.go:195,229`). If the container was built
before the `ov cdp|wl|dbus|vnc` → `ov test <verb>` move, its in-container
`ov` doesn't know the new subcommand path and the delegation fails. Fix
by rebuilding and redeploying any image that bakes `ov` (grep `image.yml`
for `- ov$` to find them). Test runner itself is unaffected — this only
bites the host→container delegation paths.

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

### Cross-distro package names (`package_map:`)

The `package:` verb chains `rpm -q || dpkg -s || pacman -Q` with a single
literal name, so a test that works on Fedora (`openssh-server`) fails on
Arch (where the package is just `openssh`). `package_map:` is the
authoring hook: the first key that matches any of the image's
`distro:` tags wins; otherwise the `package:` scalar is used as the
fallback. Source: `ov/testrun_verbs.go:resolvePackageName` + the
`Runner.Distros` field wired from `meta.Distro` at both test entry
points.

```yaml
# layers/sshd/layer.yml — cross-distro authoring
- id: openssh-server-package
  package: openssh-server              # Fedora / Debian default
  package_map:
    archlinux: openssh                 # Arch ships the metapackage as 'openssh'
    fedora: openssh-server
    fedora:43: openssh-server          # explicit version tag — matches before 'fedora'
  installed: true
```

Tag priority: entries in `distro:` are in priority order (e.g.
`["fedora:43", fedora]`), so `package_map:` can be keyed on either the
version-specific or the family tag, and the more-specific tag wins
naturally. An empty-string map value falls through to the next tag
(see `TestResolvePackageName` in `ov/testrun_verbs_test.go`).

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
| `cdp` | Method name (status/list/eval/text/html/url/axtree/screenshot/open/click/type/raw/wait/coords + spa-*) + method-specific modifiers (`tab`, `expression`, `url`, `selector`, `text`, `artifact`, `x`, `y`) + shared `stdout`/`stderr`/`exit_status`/`artifact_min_bytes` | **Deploy-scope only.** Wraps `ov test cdp <method>`. See Live-container verb catalog below. |
| `wl` | Method name (screenshot/status/click/type/key/key-combo/mouse/scroll/drag/clipboard/toplevel/windows/focus/close/geometry/xprop/atspi/exec/resolution + overlay-*/sway-*) + method-specific modifiers (`x`, `y`, `text`, `key`, `combo`, `target`, `action`, `artifact`) + shared matchers | **Deploy-scope only.** Wraps `ov test wl <method>` (including `sway` and `overlay` nested subgroups). See Live-container verb catalog below. |
| `dbus` | Method name (list/call/introspect/notify) + method-specific modifiers (`dest`, `path`, `method`, `args`, `text`) + shared matchers | **Deploy-scope only.** Wraps `ov test dbus <method>`. |
| `vnc` | Method name (status/screenshot/click/mouse/type/key/rfb/passwd) + method-specific modifiers (`x`, `y`, `text`, `key`, `artifact`) + shared matchers | **Deploy-scope only.** Wraps `ov test vnc <method>`. |
| `mcp` | Method name (ping/servers/list-tools/list-resources/list-prompts/call/read) + method-specific modifiers (`tool`, `uri`, `input`, `mcp_name`) + shared matchers | **Deploy-scope only.** Speaks `github.com/modelcontextprotocol/go-sdk` to any `mcp_provides` endpoint. See "Method allowlist — mcp" below. |

### Shared modifiers

| Field | Purpose |
|-------|---------|
| `id` | Optional stable identifier. Enables `deploy.yml` to override by `id`. Unique per section per image. |
| `description` | Human-readable label for reports. |
| `skip: true` | Always skip this check (reported but doesn't fail the run). |
| `exclude_distros: [<tag>, ...]` | Skip the check when any of the image's `distro:` tags matches an entry. Use for probes that only apply on some distros (e.g. `file: /usr/bin/fastfetch` is valid on Fedora/Arch/Debian but fastfetch is dropped from Ubuntu 24.04's noble main). Matched against the image's full distro list (`["ubuntu:24.04", "ubuntu", "debian"]`), so either `ubuntu:24.04` or `ubuntu` matches. Wired 2026-04 — see `ov/testspec.go:Check.ExcludeDistros` and `ov/testrun.go:runOne`. |
| `timeout: "5s"` | Per-check timeout (http, addr). |
| `scope: build\|deploy` | Default `build` at layer/image level, `deploy` at deploy level. |

#### `exclude_distros:` worked example

```yaml
- id: fastfetch-binary
  file: /usr/bin/fastfetch
  exists: true
  exclude_distros:
    - ubuntu:24.04
```

On `ghcr.io/overthinkos/ubuntu-coder:latest` this reports `skipped (excluded on distro "ubuntu:24.04")` instead of `failed`. On any other image it runs normally. Prefer this over dropping the test entirely — it keeps the guard in place for Fedora/Arch/Debian and documents why Ubuntu is special.

**Compared to `package_map:`** — `package_map:` changes what the test *probes for* per distro (same semantic, different package name). `exclude_distros:` skips the check entirely — use it when the functionality genuinely doesn't exist on a given distro.

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

### Live-container verb catalog: `cdp`, `wl`, `dbus`, `vnc`, `mcp`

These five verbs wrap the corresponding `ov test <verb> <method>` CLI
subcommands so every live-container operation (browser automation, Wayland
input/screenshot, D-Bus calls, VNC framebuffer capture, MCP protocol
probes) is authorable as a declarative check. All five are **deploy-scope
only** — they need a running container with port mappings; `ov image
validate` rejects them in build scope, and `ov image test` skips them at
runtime with a clear message.

**Assertion semantics:** subprocess delegation — the runner executes
`ov test <verb> <method> <image> <args…> [-i <instance>]` on the host,
captures stdout/stderr/exit, and feeds the output through the existing
matcher pipeline. Queries (status/list/eval/text/html/url/axtree/screenshot/…)
produce assertable output. Side-effect actions (click/type/open/close/…) pass
when they exit 0 — follow them with a query check to verify the effect.

**No state chaining (Phase 1):** there's no framework-managed variable to
carry a tab ID or window handle from one check to the next. Authors rely
on conventions (new tabs are usually ID 1) and capture state manually via
follow-up `cdp: list` checks if needed. Phase 2 (deferred) will add
`capture:` + `${CAPTURED:name}` for stateful flows.

#### Method allowlist — `cdp` (15 methods + 6 SPA-nested)

Queries: `status`, `list`, `url`, `text`, `html`, `eval`, `axtree`,
`coords`, `raw`, `wait`, `screenshot`.
Actions: `open`, `close`, `click`, `type`.
SPA-nested (selkies coordinate-scaling / passthrough input):
`spa-status`, `spa-click`, `spa-type`, `spa-key`, `spa-key-combo`,
`spa-mouse`.

```yaml
- id: cdp-up
  cdp: status
  stdout:
    equals: ok

- id: cdp-page-title
  cdp: eval
  tab: "1"
  expression: "document.title"
  stdout: "Dashboard"

- id: cdp-screenshot-valid
  cdp: screenshot
  tab: "1"
  artifact: /tmp/cdp.png
  artifact_min_bytes: 10000       # PNG must be non-empty
```

#### Method allowlist — `wl` (22 top-level + 4 overlay + 12 sway)

Queries: `screenshot`, `status`, `toplevel`, `windows`, `geometry`,
`xprop`, `atspi`, `clipboard`.
Actions: `click`, `double-click`, `mouse`, `scroll`, `drag`, `type`,
`key`, `key-combo`, `focus`, `close`, `fullscreen`, `minimize`, `exec`,
`resolution`.
Overlay-nested: `overlay-list`, `overlay-status`, `overlay-show`,
`overlay-hide`.
Sway-nested: `sway-tree`, `sway-workspaces`, `sway-outputs`, `sway-msg`,
`sway-focus`, `sway-move`, `sway-resize`, `sway-kill`, `sway-floating`,
`sway-layout`, `sway-workspace`, `sway-reload`.

```yaml
- id: wl-desktop-captured
  wl: screenshot
  artifact: /tmp/wl.png
  artifact_min_bytes: 10000

- id: wl-has-windows
  wl: toplevel
  stdout:
    matches: "."        # at least one line of output

- id: wl-sway-has-workspace-1
  wl: sway-workspaces
  stdout:
    contains: '"name":"1"'
```

#### Method allowlist — `dbus` (4 methods)

Queries: `list`, `call`, `introspect`.
Actions: `notify`.

```yaml
- id: dbus-notifications-registered
  dbus: list
  stdout:
    contains: "org.freedesktop.Notifications"

- id: dbus-get-capabilities
  dbus: call
  dest: org.freedesktop.Notifications
  path: /org/freedesktop/Notifications
  method: org.freedesktop.Notifications.GetCapabilities
  args: []
  stdout:
    contains: body
```

#### Method allowlist — `vnc` (8 methods)

Queries: `status`, `screenshot`, `rfb`.
Actions: `click`, `mouse`, `type`, `key`, `passwd`.

```yaml
- id: vnc-up
  vnc: status
  stdout:
    equals: ok

- id: vnc-framebuffer-captured
  vnc: screenshot
  artifact: /tmp/vnc.png
  artifact_min_bytes: 5000
```

#### Method allowlist — `mcp` (7 methods)

Queries: `ping`, `servers`, `list-tools`, `list-resources`, `list-prompts`,
`read`.
Actions: `call`.

The `mcp` verb speaks the Model Context Protocol to any server advertised
via `mcp_provides`. URL resolution is automatic — the runner reads the
image's `org.overthinkos.mcp_provides` OCI label, substitutes
`{{.ContainerName}}`, runs the entries through `podAwareMCPProvides`, and
then rewrites the host portion to `127.0.0.1:<published-host-port>` using
the same port-mapping data that powers `${HOST_PORT:N}`. No URL argument
is needed in YAML.

Transport dispatch: `transport: http` (or empty) → Streamable HTTP;
`transport: sse` → SSE. Anything else is rejected at dial time.

Disambiguation: if an image declares multiple `mcp_provides` entries,
add `mcp_name: <server>` on the check; otherwise the single entry
auto-picks.

```yaml
- id: mcp-ping
  scope: deploy
  mcp: ping
  timeout: 10s                # optional; default 30s

- id: mcp-list-tools
  scope: deploy
  mcp: list-tools
  stdout:
    - contains: insert_cell   # plaintext "name\tdescription" per line
    - contains: execute_cell

- id: mcp-call-tool
  scope: deploy
  mcp: call
  tool: list_notebooks        # required
  input: "{}"                 # optional JSON arg blob
  exit_status: 0              # assert no IsError

- id: mcp-read-resource
  scope: deploy
  mcp: read
  uri: file:///tmp/example.txt
  stdout:
    - matches: "."            # non-empty body
```

Ad-hoc CLI equivalents (each leaf also supports `--json` for programmatic
use and `--name <server>` for multi-server images):

```bash
ov test mcp ping       <image> [-i <instance>] [--name <server>]
ov test mcp servers    <image> [-i <instance>]
ov test mcp list-tools <image> [-i <instance>] [--name <server>]
ov test mcp call       <image> <tool> '<json-args>' [-i <instance>]
ov test mcp read       <image> <uri>            [-i <instance>]
```

Output format: tab-separated plaintext (one record per line) so matchers
can `contains:` without JSON decoding. `list-tools` emits
`<name>\t<description>`; `list-resources` emits `<uri>\t<name>\t<mime>`;
`call` emits the concatenated `TextContent` payloads; `ping` emits `ok`.

#### New modifier: `artifact_min_bytes`

For screenshot-producing methods (`cdp: screenshot`, `wl: screenshot`,
`vnc: screenshot`), set `artifact: <path>` to tell `ov` where to write
the image AND `artifact_min_bytes: <N>` to assert the file is at least N
bytes after the run. Guards against the common failure mode of "the
screenshot ran but produced a zero-byte file because the compositor
wasn't ready yet."

#### Gotcha — stale container-baked `ov` binary

The `dbus:` verb invokes the container's `ov` binary via delegation. If
the container image was built before the `ov cdp|wl|dbus|vnc` → `ov test
<verb>` move, its in-container `ov` expects the old paths and the
delegation fails. Rebuild and redeploy any image that bakes `ov` (grep
`image.yml` for `- ov$`). The test runner itself is unaffected; this only
bites the host→container delegation paths in the `dbus` verb.

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

### 5. Know which stream a `--version`-style command writes to

`ov version` writes to **stdout** — the canonical `layers/ov/layer.yml`
test asserts a `stdout:` matcher. (Prior versions used Go's builtin
`println(version)` which routes to stderr; the fix to `fmt.Println`
landed alongside the MCP server work so every call stays captured.)

```yaml
- id: ov-version
  command: /usr/local/bin/ov version
  exit_status: 0
  stdout:
    - matches: "[0-9]{4}\\.[0-9]+"
```

**But other tools genuinely write to stderr** — `ssh -V`, `openssl version`,
some `*-config --version` scripts. Always probe first
(`ov cmd <image> '<cmd> 2>&1 >/dev/null'` — if the output shows up here
it was stderr; if silent, it's stdout). See also the `ssh-version`
check in `layers/ssh-client/layer.yml` which IS a real stderr case.

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
directive (typically uid 1000 for containers). Root-only files
(e.g. `/etc/sudoers.d/*` is root:root 0750) report as "missing" because
the user can't traverse the parent directory.

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

### 11. **Bootc images keep USER=root** — tests must cover both modes

Gotcha 10 flips for bootc. `ov/generate.go` deliberately omits the final `USER <uid>` directive on bootc images because systemd (PID 1 in a bootc VM) manages user sessions via login, so the container's own USER directive is irrelevant. Result: the same `ov image test` that runs as uid 1000 in a container runs as **uid 0** in a bootc image.

That breaks any test that assumes the user-context default. A `sudo -n -l` check that expects `NOPASSWD` in the output works for non-bootc (USER=1000 → sudo lists the user's NOPASSWD rule) but fails for bootc (USER=0 → sudo prints root's Defaults block, which doesn't contain the literal string `NOPASSWD`). Same failure mode for any `test -f ~/.config/foo` that depends on `$HOME=/home/user`.

The fix: drop into `user` explicitly when running as root, stay as-is otherwise. Use **`runuser -u user -- <cmd>`** as the bridge — it's portable across util-linux versions and preserves stdout:

```yaml
# Dual-mode: container (USER=1000) AND bootc (USER=root) both pass
- id: sudoers-ov-user
  command: |
    if [ "$(id -u)" = "0" ]; then
      runuser -u user -- sudo -n -l
    else
      sudo -n -l
    fi
  exit_status: 0
  stdout:
    - contains: "NOPASSWD"
```

**Don't use `runuser -l user -s /bin/bash -c '…'`.** On Arch's util-linux
(2.42+), `runuser -l … -c` swallows the wrapped command's stdout when
the shell is a login shell — reproduced: `runuser -l user -s /bin/bash
-c 'sudo -n -l'` prints nothing and exits 0; `runuser -u user -- sudo
-n -l` prints the full NOPASSWD listing. This bit the earlier shipping
form of the test; the sshd layer now uses `-u … --`.

Worked example: `/ov-layers:sshd` ships exactly the `-u user --` pattern. Alternative: make the check `scope: deploy` so it runs against a live container where you control the user-switch externally.

### 12. **`ov image test <short-name>` is ambiguous with multiple CalVer tags** — use the full registry ref

`ov image test` resolves its positional argument against **local podman storage**, not `image.yml`. When the host has accumulated many CalVer tags for the same image (a normal consequence of iterative `ov image build` runs), the short form errors out:

```
ov: error: ambiguous short name "selkies-desktop-ov" in local storage;
           candidates: ghcr.io/overthinkos/selkies-desktop-ov:latest,
           ghcr.io/overthinkos/selkies-desktop-ov:2026.109.1418,
           ... Re-run with a full ref.
```

Use the fully-qualified registry ref:

```bash
ov image test ghcr.io/overthinkos/selkies-desktop-ov:latest
```

This is different from `ov image inspect`, `ov image build`, and `ov test` (the live-service runner), which key off `image.yml` and accept short names unambiguously. Only the disposable-container runner has this restriction because it does not consult `image.yml` at all.

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
| `jupyter` | 32 | 32 | 0 | 0 (includes 3 declarative `mcp:` checks added Apr 2026) |
| `openwebui` | 24 | 24 | 0 | 0 |
| `hermes` | 50 | 50 | 0 | 0 |
| `immich-ml` | 63 | 61 | 0 | 2 (redis port internal) |
| `selkies-desktop` | 91 | 91 | 0 | 0 |
| `sway-browser-vnc` | 92 | 92 | 0 | 0 (includes 7 declarative cdp/wl/dbus/vnc/mcp checks; 2 mcp tests come from the chrome-devtools-mcp layer) |

**Total: 376 checks across 7 images (0 failing).**

The "skipped" entries are intentional — they reference ports that
aren't mapped in the containing image's `ports:` block. Skipping them
is correct behavior; they'd pass in an image that does expose those
ports.

## Related skills

- **Nested verbs under `ov test`** — `/ov:cdp`, `/ov:wl`, `/ov:dbus`, `/ov:vnc`, `/ov:mcp` are dispatched as `ov test cdp|wl|dbus|vnc|mcp`. See the Subcommands section above.
- `/ov:layer` — layer authoring; `tests:` field is part of every `layer.yml`.
- `/ov:image` — image-level `tests:` and `deploy_tests:` at composition time.
- `/ov:deploy` — local `deploy.yml` overlay rules and the `tests:` merge.
- `/ov:validate` — static schema + cross-scope variable checks; the first
  gate before `ov image build`.
- `/ov:build` — how tests are embedded into the OCI label at build time;
  LABELs-at-end cache behavior.
- `/ov:inspect` — view the merged 3-section tests structure as JSON.
- `/ov-dev:go` — implementation map: `testspec.go`, `testvars.go`,
  `testrun.go`, `testrun_verbs.go`, `testrun_ov_verbs.go`, `testcollect.go`,
  `test_cmd.go`, `validate_tests.go`, `mcp.go`, `mcp_client.go`, plus the
  `LabelTests` constant in `labels.go`.
- `/ov-dev:generate` — how `LabelTests` is written into the Containerfile
  via `writeJSONLabel`, and why the LABEL block lives at the end of the
  final stage.

## When to Use This Skill

**MUST be invoked** before authoring, running, or debugging declarative
tests at any level. If the user mentions `ov test`, `ov image test`,
`tests:`, `deploy_tests:`, `LabelTests`, or any check verb by name
(file/port/http/command/package/service/...), invoke this skill first.
