---
name: appium
description: |
  MUST be invoked before any work involving: ov eval appium commands, Android UI automation, WebDriver session management, APK install via Appium, element find/click/send-keys, mobile-specific WebDriver caps — anywhere the goal is to drive a running Appium 3.x server inside a container via W3C WebDriver from outside.
---

# Appium — W3C WebDriver client

## Overview

`ov eval appium <method>` is the host-side Appium WebDriver client. It
talks W3C WebDriver to the container's host-published Appium port
(container `:4723` → host's `HOST_PORT:4723`, e.g. `35001` on the
`eval-android-emulator-pod` deploy) at base path `/wd/hub`.

Session lifecycle uses a persistent JSON session file at
`~/.cache/ov/appium/sessions/<image>[_<instance>].json` so multi-step
tests share one WebDriver session across separate `ov eval appium`
invocations. `session-create` writes the file; the other operations
load it; `session-delete` removes it and best-effort closes the remote
session.

The Appium-specific endpoints not in the standard WebDriver surface
(`install_app`, `is_app_installed`, `remove_app`, etc.) route through
the W3C escape hatch: `POST /session/<id>/execute/sync` with
`{"script": "mobile: installApp", "args": [...]}`.

### Also as a declarative verb

Every `ov eval appium <method>` is authorable as an `appium:` verb
inside an `eval:` block. **Deploy-scope only**.

```yaml
- id: appium-up
  scope: deploy
  appium: status
  stdout: { contains: '"ready":true' }
- id: open-session
  scope: deploy
  appium: session-create
  caps: |
    {"platformName":"Android","appium:automationName":"UiAutomator2","appium:deviceName":"emulator-5554"}
- id: install
  scope: deploy
  appium: install-app
  apk: ./tests/data/ApiDemos-debug.apk   # HOST path; staged into the container by the verb
- id: tap-animation
  scope: deploy
  appium: click
  strategy: xpath
  selector: '//android.widget.TextView[@text="Animation"]'
- id: snapshot
  scope: deploy
  appium: screenshot
  artifact: /tmp/post-tap.png
  artifact_min_bytes: 10000
- id: close
  scope: deploy
  appium: session-delete
```

## Quick Reference

| Subcommand | CLI form | Required modifier | Description |
|---|---|---|---|
| `status` | `ov eval appium status <image>` | — | GET /status, prints JSON, fails on HTTP != 200 |
| `session-create` | `ov eval appium session-create <image> --caps <json>` | `caps:` | Create W3C session, persist id |
| `session-delete` | `ov eval appium session-delete <image>` | — | Close session and remove file |
| `install-app` | `ov eval appium install-app <image> --apk <path>` | `apk:` | mobile:installApp escape hatch |
| `find` | `ov eval appium find <image> --selector <s> [--strategy STRAT]` | `selector:` | Find element, prints W3C id |
| `click` | `ov eval appium click <image> --selector <s>` | `selector:` | Find + click |
| `send-keys` | `ov eval appium send-keys <image> --selector <s> --text <t>` | `selector:`+`text:` | Find + type |
| `screenshot` | `ov eval appium screenshot <image> --artifact <png>` | `artifact:` | GET /screenshot, decode base64, write PNG |

`--strategy` accepts: `xpath` (default), `id`, `accessibility-id`,
`class-name`, `android-uiautomator`, `name`, `css`. Mapped to W3C / Appium
locator strings inside the implementation.

## Two critical gotchas

### W3C capabilities (not legacy JSONWire)

Appium 3.x rejects the legacy JSONWire capability format with HTTP 400.
**Always use W3C-style caps**:

```yaml
caps: |
  {"platformName":"Android",
   "appium:automationName":"UiAutomator2",
   "appium:deviceName":"emulator-5554"}
```

The `appium:` vendor prefix is mandatory on Appium-specific keys
(`automationName`, `deviceName`, `app`, `appPackage`, `appActivity`,
`noReset`, `newCommandTimeout`, …). Plain W3C keys (`platformName`,
`browserName`) have no prefix.

The selenium SDK we use (`github.com/tebeka/selenium`) handles the
`alwaysMatch` wrapping internally — pass flat caps and the SDK wraps.
We unwrap the user's `alwaysMatch` key if present to avoid double-
wrapping.

### `apk:` is a HOST path for BOTH adb and appium

- `adb: install` reads `apk:` from the host filesystem (the host `ov`
  binary pushes via the ADB sync protocol).
- `appium: install-app` ALSO reads `apk:` from the host filesystem. Because
  the in-container Appium server's `mobile: installApp` requires an
  `appPath` it can read (the base64 `{"app": …}` form is rejected with HTTP
  400 "required parameter is missing: appPath"), the verb stages the host
  APK INTO the container via `<engine> cp` to a temp path, calls installApp
  with that in-container path, then removes the temp file. No bind-mount and
  no external staging step are needed — `apk:` is the host path, end to end.

Both install verbs are therefore symmetric: `apk:` is always a host path
(typically `./tests/data/<app>.apk`, resolved against the project root).
If `appium: install-app` fails, check the host path exists, not a container
path.

## Method allowlist + required modifiers

| Method | Required modifiers | Notes |
|---|---|---|
| `status` | — | Bypasses the SDK — plain `http.Get` against `/status`. |
| `session-create` | `Caps` | Accepts both flat `{"k":"v"}` and pre-wrapped `{"alwaysMatch":{"k":"v"}}`. Use `--caps @path.json` to read from file. Deletes any pre-existing session for the same image+instance first (best-effort), then `selenium.NewRemote` + persist. |
| `session-delete` | — | Best-effort `DELETE /session/<id>` + `rm` of session file. No-op if no session exists. |
| `install-app` | `Apk` | `Apk` is a HOST path. The verb stages it into the container via `<engine> cp`, then `POST /session/<id>/execute/sync` with `mobile: installApp` + `{appPath: <in-container-temp>}`, then removes the temp file. Symmetric with `adb: install`. |
| `find` | `Selector` | `POST /session/<id>/element`, prints the W3C element id (a UUID-ish string). |
| `click` | `Selector` | Find + `POST /session/<id>/element/<eid>/click`. Atomic. |
| `send-keys` | `Selector`+`Text` | Find + `POST /session/<id>/element/<eid>/value` with `{text: <Text>}`. |
| `screenshot` | `Artifact` | `GET /session/<id>/screenshot`, base64-decode, write to `Artifact`. Pairs with `artifact_min_bytes:`. |

## Session-file format + location

`~/.cache/ov/appium/sessions/<image>[_<instance>].json` (XDG-cache; honours
`XDG_CACHE_HOME`):

```json
{
  "session_id": "37e8f3c1-a9b2-4d8e-b6c5-9a4f7c8b1e2d",
  "base_url": "http://127.0.0.1:35001/wd/hub",
  "created_at": "2025-01-15T14:32:17.481Z",
  "image": "eval-android-emulator-pod",
  "instance": "",
  "caps": { ... }
}
```

Mode `0600`. Why XDG cache and NOT in-project / NOT `~/.local/share`:

- Session ids are host-local ephemeral state. Different hosts running
  the same image have different containers with different session ids;
  sharing the file (e.g. via Syncthing of an in-project path) would
  corrupt cross-host setups.
- `~/.local/share` is XDG **data** — explicitly Syncthing-replicated on
  this user's hosts. XDG **cache** is the correct location for ephemeral
  host-local state.

Override the session id for a single check with `session:`:

```yaml
- appium: screenshot
  artifact: /tmp/x.png
  session: 37e8f3c1-...   # bypass the session file
```

## Implementation

- `ov/appium.go` — CLI subcommands + Run() methods. `session-create` uses
  `github.com/tebeka/selenium`'s `NewRemote` (which handles W3C
  `alwaysMatch` wrapping). All other ops use a small raw-HTTP W3C client
  (`w3cSession`) because the SDK can't attach to an existing session id.
- `ov/appium_session.go` — session-file load/save/delete + XDG path
  resolution.
- `ov/evalrun_ov_verbs.go` — `appiumMethods` allowlist + `runAppium`
  dispatcher.
- `ov/validate_eval.go` — `case "appium"` in `validateOvVerb`.

The host port is read from podman's `NetworkSettings.Ports` via
`InspectContainer` — same path as the adb verb.

## Library risks (documented)

`github.com/tebeka/selenium` last released v0.9.9 in 2022. The W3C
WebDriver protocol is stable so it works against Appium 3.x today, but
upstream activity is low. If it goes dormant we'll need to fork or
migrate to a newer library. The blast radius is small — only
`session-create` uses the SDK; the rest go through `w3cSession` (plain
HTTP), so an SDK swap is a single Run() method's worth of code.

`github.com/zach-klippenstein/goadb` (used by the sibling `/ov-eval:adb`)
has the same maintenance posture — pinned to a 2020 release. Same
mitigation if it stops working.

## Related skills

- `/ov-eval:adb` — sibling verb for low-level Android Debug Bridge
  control (install / shell / screencap / logcat).
- `/ov-eval:eval` — the unified eval system and the Check struct that
  holds every verb discriminator + modifier.
- `/ov-tools:android-emulator` (when authored) — the image these verbs target.

## When to Use This Skill

**MUST be invoked** for any task involving `ov eval appium` commands or
`appium:` declarative checks. Invoke this skill BEFORE reading the
verb's Go source or reaching for `command: curl http://localhost:.../wd/hub/...`
workarounds.
