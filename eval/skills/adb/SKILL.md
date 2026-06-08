---
name: adb
description: |
  MUST be invoked before any work involving: charly eval adb commands, Android Debug Bridge interaction, APK install/uninstall, device shell command execution, system property reads, screencap, logcat tailing — anywhere the goal is to drive a running Android emulator from outside the container via the host-published ADB server port.
---

# ADB — Android Debug Bridge

## Overview

`charly eval adb <method>` is the host-side ADB client. The host `ov` binary
connects to the running container's host-published ADB server port
(container `:5037` → host's `HOST_PORT:5037`, e.g. `35002` on the
`eval-android-emulator-pod` deploy) using `github.com/zach-klippenstein/goadb`
and dispatches operations against the emulator backing that server.

Same architecture pattern as `charly eval mcp`: host-side protocol client, no
container-side helper, works against any deploy that publishes the
adb-server port — pod / vm / host / nested all work transparently because
the connection is plain TCP to `127.0.0.1:<host-port>` and the
portforward / passt / etc. layer handles the rest.

### Also as a declarative verb

Every `charly eval adb <method>` is authorable as an `adb:` verb inside a
`eval:` block. The method name becomes the verb's YAML value; method-
specific args are sibling fields (`apk:`, `property:`, `args:`,
`artifact:`). Shared matchers (`stdout:`, `stderr:`, `exit_status:`,
`artifact_min_bytes:`) work like other verbs. **Deploy-scope only** —
needs a running container with a host-mapped ADB server port; the
validator rejects build-scope use at `charly box validate` time.

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

See `/charly-eval:eval` for the full per-verb schema notes; this skill is the
adb-specific method reference.

## Quick Reference

| Subcommand | CLI form | Required modifier | Description |
|---|---|---|---|
| `devices` | `charly eval adb devices <image>` | — | List devices/emulators with state |
| `shell` | `charly eval adb shell <image> -- <cmd...>` | `args:` (list) | Run a shell command on the emulator |
| `install` | `charly eval adb install <image> --apk <path>` | `apk:` | Install an APK from the host filesystem |
| `uninstall` | `charly eval adb uninstall <image> <package>` | `args: [pkgid]` | Remove a package by id |
| `getprop` | `charly eval adb getprop <image> <property>` | `property:` | Read a system property |
| `screencap` | `charly eval adb screencap <image> --artifact <png>` | `artifact:` | Capture a PNG screenshot to a host file |
| `logcat-tail` | `charly eval adb logcat-tail <image> [--lines N] [--filter F]` | — | Dump recent logcat lines (uses `logcat -d`) |
| `wait-for-device` | `charly eval adb wait-for-device <image> [--timeout 60s]` | — | Block until `sys.boot_completed=1` |
| `wait-ui-settled` | `charly eval adb wait-ui-settled <image> [--timeout 600s]` | — | Block until the foreground is not an ANR dialog (dismissing via HOME) |
| `current-focus` | `charly eval adb current-focus <image>` | — | Print the foreground window line (`mCurrentFocus`) |
| `keyevent` | `charly eval adb keyevent <image> <key>` | `key:` (arg) | Send a key event (`input keyevent KEYCODE_…`) |

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
| `wait-ui-settled` | — | Polls `mCurrentFocus` (`dumpsys window`); while a system "Application Not Responding" dialog holds focus it dismisses it with `KEYCODE_HOME` and keeps polling, up to `Timeout`. Prints `settled` on success. **Set `timeout:` generously** (the android-emulator gate uses 600s) — the shared 30s default is too short for a churning emulator. Pure goadb — no in-container shell. |
| `current-focus` | — | Prints the `mCurrentFocus` window line. Assert the foreground app (`stdout: contains: io.appium.android.apis`) or detect a stuck ANR dialog. |
| `keyevent` | `KeyName` | `input keyevent <key>` via `key: KEYCODE_HOME` / `KEYCODE_BACK` / a numeric code. Generic input building block; `wait-ui-settled` uses the same call internally to dismiss dialogs. |

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

### `wait-ui-settled` — UI readiness past `sys.boot_completed`

A freshly-booted `google_apis_playstore` emulator runs minutes of GMS
post-boot churn (Play Store auto-update, Chimera config, Heterodyne sync)
that starves the GMS-coupled system UI (Pixel Launcher via AiAi,
systemui). It ANRs, and the "Application Not Responding" dialog occludes
whatever app is foreground — so a UI probe (appium find, `monkey`)
fired right after `sys.boot_completed` silently fails. `sys.boot_completed`
is necessary-but-not-sufficient readiness; `wait-ui-settled` is the
sufficient half: it polls the focused window and dismisses any ANR dialog
with `KEYCODE_HOME` until the foreground is clean. It is **load-independent**
(the ANR is GMS churn, not host load) and adapts to a load-dilated churn
purely via `timeout:`. Pure Go over goadb — no in-container shell, so it
is immune to the stdin-heredoc hazard that breaks shell-based settle loops
(see `/charly-eval:eval` "in-container `command:` stdin").

```yaml
# Order matters — wait-for-device (boot) BEFORE wait-ui-settled (UI),
# wait-ui-settled BEFORE any UI-interacting check (appium / monkey).
- id: emulator-ui-settled
  scope: deploy
  adb: wait-ui-settled
  timeout: 600s
  stdout:
    - contains: settled
```

`current-focus` and `keyevent` are the building blocks `wait-ui-settled`
composes — usable on their own to assert the foreground app or inject a
key (`adb: keyevent` + `key: KEYCODE_BACK`).

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

- `/charly-eval:appium` — sibling verb for higher-level UI automation against the
  same emulator (W3C WebDriver).
- `/charly-eval:android` — the `kind: android` device + `apk:` package format +
  `target: android` deploy. `charly eval adb install` / `install-app` are thin
  wrappers over the SAME shared installer (`ov/android_install.go`) the apk
  format drives — so the verb, the format, and the deploy target never drift.
- `/charly-eval:eval` — the unified eval system and the Check struct that holds
  every verb discriminator + modifier.
- `/charly-tools:android-emulator` (when authored) — the image these verbs target.

## When to Use This Skill

**MUST be invoked** for any task involving `charly eval adb` commands or
`adb:` declarative checks. Invoke this skill BEFORE reading the verb's
Go source or reaching for `command: adb ...` workarounds.
