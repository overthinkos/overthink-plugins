---
name: adb
description: |
  MUST be invoked before any work involving: ov eval adb commands, Android Debug Bridge interaction, APK install/uninstall, device shell command execution, system property reads, screencap, logcat tailing — anywhere the goal is to drive a running Android emulator from outside the container via the host-published ADB server port.
---

# ADB — Android Debug Bridge

## Overview

`ov eval adb <method>` is the host-side ADB client. The host `ov` binary
connects to the running container's host-published ADB server port
(container `:5037` → host's `HOST_PORT:5037`, e.g. `35002` on the
`eval-android-emulator-pod` deploy) using `github.com/zach-klippenstein/goadb`
and dispatches operations against the emulator backing that server.

Same architecture pattern as `ov eval mcp`: host-side protocol client, no
container-side helper, works against any deploy that publishes the
adb-server port — pod / vm / host / nested all work transparently because
the connection is plain TCP to `127.0.0.1:<host-port>` and the
portforward / passt / etc. layer handles the rest.

### Also as a declarative verb

Every `ov eval adb <method>` is authorable as an `adb:` verb inside a
`eval:` block. The method name becomes the verb's YAML value; method-
specific args are sibling fields (`apk:`, `property:`, `args:`,
`artifact:`). Shared matchers (`stdout:`, `stderr:`, `exit_status:`,
`artifact_min_bytes:`) work like other verbs. **Deploy-scope only** —
needs a running container with a host-mapped ADB server port; the
validator rejects build-scope use at `ov image validate` time.

Example:

```yaml
- id: emulator-is-up
  scope: deploy
  adb: devices
  stdout:
    - contains: "emulator-5554"
- id: boot-done
  scope: deploy
  adb: getprop
  property: sys.boot_completed
  stdout:
    - contains: "1"
  timeout: 300s
```

See `/ov-eval:eval` for the full per-verb schema notes; this skill is the
adb-specific method reference.

## Quick Reference

| Subcommand | CLI form | Required modifier | Description |
|---|---|---|---|
| `devices` | `ov eval adb devices <image>` | — | List devices/emulators with state |
| `shell` | `ov eval adb shell <image> -- <cmd...>` | `args:` (list) | Run a shell command on the emulator |
| `install` | `ov eval adb install <image> --apk <path>` | `apk:` | Install an APK from the host filesystem |
| `uninstall` | `ov eval adb uninstall <image> <package>` | `args: [pkgid]` | Remove a package by id |
| `getprop` | `ov eval adb getprop <image> <property>` | `property:` | Read a system property |
| `screencap` | `ov eval adb screencap <image> --artifact <png>` | `artifact:` | Capture a PNG screenshot to a host file |
| `logcat-tail` | `ov eval adb logcat-tail <image> [--lines N] [--filter F]` | — | Dump recent logcat lines (uses `logcat -d`) |
| `wait-for-device` | `ov eval adb wait-for-device <image> [--timeout 60s]` | — | Block until `sys.boot_completed=1` |

`--serial <serial>` selects a specific device (default `emulator-5554`).
`-i <instance>` addresses a `<base>/<instance>` pod deploy.

## Method allowlist + required modifiers

| Method | Required modifiers | Notes |
|---|---|---|
| `devices` | — | Output is one line per device: `<serial>\t<state>` |
| `shell` | `Args` | `Args[0]` is the program; `Args[1:]` are positional args. Prefix `--` is auto-added so flags don't leak into the outer Kong invocation. |
| `install` | `Apk` | Pushes the APK via the sync protocol to `/data/local/tmp/`, then `pm install -r`. Best-effort cleanup of the staged file. Asserts `Success` in pm output. |
| `uninstall` | `Args` | `Args[0]` = package id (e.g. `com.example.android.apis`). Same `Args:[0]` convention as `shell` to keep the modifier surface flat. |
| `getprop` | `Property` | Property key like `sys.boot_completed`, `ro.build.version.release`. |
| `screencap` | `Artifact` | Runs `screencap -p \| base64` on the device, decodes host-side, writes to `Artifact`. The `base64` round-trip is mandatory — raw binary stdout gets mangled by goadb's shell stream on some emulator builds. Pairs with `artifact_min_bytes:` for size assertion. |
| `logcat-tail` | — | Uses `logcat -d` (dump-and-exit). `Amount` (`--lines`) trims to the last N lines host-side; `Query` (`--filter`) is the literal filter spec (e.g. `MyApp:I *:S`). |
| `wait-for-device` | — | Polls `getprop sys.boot_completed` every 2s up to `Timeout` (default 60s). Lighter than the wire-protocol `wait-for-device` because that returns the moment the device ATTACHES (well before sys.boot_completed). |

## Authoring gotchas

### `apk:` must be a host-side path

The host `ov` binary reads the APK from its own filesystem and pushes
via the sync protocol — `apk:` is a path on the host running `ov`, NOT
a container-internal path. (Compare with `appium: install-app` where
`apk:` is the in-container Appium server's view of the path because
Appium reads the file itself.)

### `args:` for shell commands; prefix `--` is automatic

Always write shell args as a YAML list:

```yaml
adb: shell
args: [pm, list, packages, io.appium.android.apis]
```

The `--` prefix is automatic — you don't have to escape `-l` / `-p` /
`-fsS` etc. flags. Don't reach for `command:` here; `command:` is a
different verb.

### `args: [pkgid]` for uninstall

Mirrors `shell`'s convention to avoid adding a dedicated `package:`
field. The validator enforces `Args` non-empty at config time.

### Host-side screencap needs `screencap -p | base64`

A raw `screencap -p` produces binary PNG bytes that goadb's shell
stream may mangle. The implementation always pipes through base64 and
decodes host-side. This is invisible to authors — `adb: screencap`
"just works" — but worth knowing when debugging.

### `--filter` is a free-form logcat spec

`--filter "MyApp:I *:S"` passes `MyApp:I *:S` to logcat verbatim (each
word becomes one positional arg). Don't quote individual words; the
shell-fields split handles separation.

### Long-running wait

`wait-for-device` blocks up to `--timeout` (default 60s) — set
`timeout:` on the eval check to a value at least as large as
`--timeout`, plus a small margin. Common pattern:

```yaml
adb: wait-for-device
timeout: 120s
```

## Implementation

Source: `ov/adb.go` (CLI subcommands + Run() methods using
`github.com/zach-klippenstein/goadb`); allowlist + dispatcher:
`ov/evalrun_ov_verbs.go` (`adbMethods` map + `runAdb`); validator:
`ov/validate_eval.go` (`case "adb"` in `validateOvVerb`).

The host port is read from podman's `NetworkSettings.Ports` via
`InspectContainer` — same source-of-truth used by the eval test
runner's `${HOST_PORT:N}` substitution. Host-networked containers
expose the container port AS the host port.

## Related skills

- `/ov-eval:appium` — sibling verb for higher-level UI automation against the
  same emulator (W3C WebDriver).
- `/ov-eval:android` — the `kind: android` device + `apk:` package format +
  `target: android` deploy. `ov eval adb install` / `install-app` are thin
  wrappers over the SAME shared installer (`ov/android_install.go`) the apk
  format drives — so the verb, the format, and the deploy target never drift.
- `/ov-eval:eval` — the unified eval system and the Check struct that holds
  every verb discriminator + modifier.
- `/ov-tools:android-emulator` (when authored) — the image these verbs target.

## When to Use This Skill

**MUST be invoked** for any task involving `ov eval adb` commands or
`adb:` declarative checks. Invoke this skill BEFORE reading the verb's
Go source or reaching for `command: adb ...` workarounds.
