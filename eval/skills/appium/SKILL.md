---
name: appium
description: |
  MUST be invoked before any work involving: charly eval appium commands, Android UI automation, WebDriver session management, APK install via Appium, element find/click/send-keys, mobile-specific WebDriver caps — anywhere the goal is to drive a running Appium 3.x server inside a container via W3C WebDriver from outside.
---

# Appium — W3C WebDriver client

## Overview

`charly eval appium <method>` is the host-side Appium WebDriver client. It
talks W3C WebDriver to the container's host-published Appium port
(container `:4723` → host's `HOST_PORT:4723`, e.g. `35001` on the
`eval-android-emulator-pod` deploy) at base path `/wd/hub`.

Session lifecycle uses a persistent JSON session file at
`~/.cache/charly/appium/sessions/<image>[_<instance>].json` so multi-step
tests share one WebDriver session across separate `charly eval appium`
invocations. `session-create` writes the file; the other operations
load it; `session-delete` removes it and best-effort closes the remote
session.

The Appium-specific endpoints not in the standard WebDriver surface
(`install_app`, `is_app_installed`, `remove_app`, etc.) route through
the W3C escape hatch: `POST /session/<id>/execute/sync` with
`{"script": "mobile: installApp", "args": [...]}`.

### Also as a declarative verb

Every `charly eval appium <method>` is authorable as an `appium:` verb
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
| `status` | `charly eval appium status <image>` | — | GET /status, prints JSON, fails on HTTP != 200 |
| `session-create` | `charly eval appium session-create <image> --caps <json>` | `caps:` | Create W3C session, persist id |
| `session-delete` | `charly eval appium session-delete <image>` | — | Close session and remove file |
| `install-app` | `charly eval appium install-app <image> --apk <path>` | `apk:` | mobile:installApp escape hatch |
| `find` | `charly eval appium find <image> --selector <s> [--strategy STRAT]` | `selector:` | Find element, prints W3C id |
| `click` | `charly eval appium click <image> --selector <s>` | `selector:` | Find + click |
| `send-keys` | `charly eval appium send-keys <image> --selector <s> --text <t>` | `selector:`+`text:` | Find + type |
| `screenshot` | `charly eval appium screenshot <image> --artifact <png>` | `artifact:` | GET /screenshot, decode base64, write PNG |
| `get-text` | `charly eval appium get-text <image> --selector <s>` | `selector:` | find + GET .../text, prints element text |
| `get-attribute` | `charly eval appium get-attribute <image> --selector <s> --attribute <a>` | `selector:`+`attribute:` | find + GET .../attribute/<a> (checked/enabled/text/...) |
| `clear` | `charly eval appium clear <image> --selector <s>` | `selector:` | find + POST .../clear |
| `find-all` | `charly eval appium find-all <image> --selector <s>` | `selector:` | POST /elements; prints count + ids |
| `source` | `charly eval appium source <image>` | — | GET /source (UI hierarchy XML) |
| `back` | `charly eval appium back <image>` | — | POST /back (navigate back) |

### Tier 2 — per-class sugar groups (nested CLI; flat `<group>-<op>` in eval YAML)

Mirrors `wl`'s `sway-*` / `overlay-*` pattern — `charly eval appium gesture tap …`
on the CLI is `appium: gesture-tap` in an `eval:` block.

| Group | Ops (eval-YAML method names) | Key modifiers |
|---|---|---|
| `gesture` | `gesture-tap`, `gesture-double-tap`, `gesture-long-press`, `gesture-drag`, `gesture-swipe`, `gesture-scroll`, `gesture-fling`, `gesture-pinch-open`, `gesture-pinch-close` | `selector`(+`strategy`) **or** `x`+`y`; `direction`+`percent` (swipe/scroll/fling); `params` (JSON: speed/duration/endX/endY/left/top/width/height) |
| `app` | `app-start-activity`, `app-activate`, `app-terminate`, `app-remove`, `app-clear`, `app-is-installed`, `app-state`, `app-current-activity`, `app-current-package` | `app_id` (lifecycle ops); `activity` (`pkg/.activity`, start-activity, intent form) |
| `key` | `key-press`, `key-hide`, `key-shown` | `keycode` (press; 4=BACK, 66=ENTER); `params` (metastate/flags) |
| `device` | `device-info`, `device-battery`, `device-time`, `device-orientation`, `device-set-orientation`, `device-notifications`, `device-get-clipboard`, `device-set-clipboard`, `device-contexts`, `device-context` | `params` (value/name for set-orientation / set-clipboard / context switch) |

### Tier 3 — generic escape hatch (the cdp-`raw` equivalent — 100% coverage)

| Method | CLI form | Required | Description |
|---|---|---|---|
| `execute` | `charly eval appium execute <image> --expression <s> [--request-body <json>]` | `expression:` | `POST /execute/sync` — any `mobile:` command or JS. Object `request_body` → `[obj]`; array → as-is. Optional `selector:` resolves an element id for `{element}` substitution in `request_body`. |
| `raw` | `charly eval appium raw <image> --method <verb> --path <p> [--request-body <json>]` | `method:`+`path:` | Any W3C call relative to `/session/<id>` (charly prepends it). `path:`/`request_body:` support the `{element}` token when `selector:` is set. Reaches **everything** including `mobile:` via `/execute/sync`. |

**`raw` is the coverage guarantee** — anything not covered by a typed method or
sugar group is reachable here (e.g. `raw GET /element/{element}/rect`,
`raw POST /timeouts {"implicit":10000}`).

`--strategy` accepts: `xpath` (default), `id`, `accessibility-id`,
`class-name`, `android-uiautomator`, `name`, `css`. Mapped to W3C / Appium
locator strings inside the implementation.

## Gotchas

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

- `adb: install` reads `apk:` from the host filesystem (the host `charly`
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

### `app-start-activity` uses the intent form (verified)

`appium: app-start-activity` with `activity: io.appium.android.apis/.view.X`
sends `mobile: startActivity {"intent":"<activity>"}`. The intent form
(`pkg/.activity`) is what works on this UiAutomator2 build (verified live) —
NOT a split `{appPackage,appActivity}`. Extra intent args go in `params:`.

### Set a W3C implicit wait, or finds race the activity

`app-start-activity` returns before the activity's UI is laid out, so a `find`
fired immediately after it returns "no such element". Set an implicit wait once
after `session-create` so every `find` polls until the element renders:

```yaml
- appium: raw
  method: POST
  path: /timeouts
  request_body: '{"implicit":10000}'
```

(Verified: without it, ~half the per-screen checks fail the layout race; with
it, all pass. The `eval-android-emulator-pod` bed does exactly this.)

### `{element}` substitution in execute/raw

`execute` / `raw` resolve an element host-side when `selector:` is set and
substitute its W3C id for the literal token `{element}` in `request_body:`
(execute) and `path:`+`request_body:` (raw), e.g.
`raw method: GET path: /element/{element}/text selector: …`. Single-brace
`{element}` (deliberately NOT `${…}`) so the eval runtime-variable resolver
leaves it untouched. `charly box validate` errors if `{element}` appears with no
`selector:`.

### Android-14 permission dialog covers the app

On API 34, ApiDemos (and many apps) raise a runtime POST_NOTIFICATIONS dialog
(`com.google.android.permissioncontroller`) on first launch that covers the
app, so every find returns "no such element". Pre-grant the permission before
driving the app: `adb: shell arg: [pm, grant, <pkg>, android.permission.POST_NOTIFICATIONS]`
then `adb: shell arg: [am, force-stop, <pkg>]` for a clean launch.

### WebView context switch needs a pinned chromedriver (NOT --allow-insecure)

The API-34 google_apis System WebView is **Chrome 113**. UiAutomator2's
chromedriver autodownload was verified slow/hanging. Instead the `appium-server`
layer pre-bakes chromedriver 113 at `/opt/chromedriver/113` and the WebView
session sets `appium:chromedriverExecutableDir:"/opt/chromedriver/113"` +
`appium:chromedriverDisableBuildCheck:true` (both plain caps, NOT insecure-
gated). `device-contexts` (presence) works with no chromedriver and is the
reliable floor; the `device-context` switch needs the pinned chromedriver.

### Base path `/wd/hub`; AUR-provisioned toolchain

The container's Appium server (`/usr/bin/appium`, from the CachyOS/AUR `appium`
package) listens at base path **`/wd/hub`**. The whole Android toolchain comes
from CachyOS/AUR packages (`appium`, `android-sdk-*`, `android-emulator`,
`android-sdk-build-tools-34` for aapt2) under `/opt/android-sdk`; only the
API-34 system image is sdkmanager-fetched (no package exists).

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
| `get-text` / `clear` / `find-all` | `Selector` | find-then-act over `/element[s]/<id>/{text,clear}` (find-all → `/elements`). |
| `get-attribute` | `Selector`+`Attribute` | find + `GET .../attribute/<name>`. |
| `source` / `back` | — | `GET /source` / `POST /back`. |
| `gesture-*` | — (element-or-xy enforced in `Run()`); `gesture-swipe`/`scroll`/`fling` need `Direction` | `mobile: <name>Gesture`; element id from `Selector`, else `X`/`Y`; `Percent`+`Params` merged into args. |
| `app-start-activity` | `Activity` | `mobile: startActivity {intent}`. |
| `app-activate`/`terminate`/`remove`/`clear`/`is-installed`/`state` | `AppId` | `mobile: <name>App {appId}`. |
| `app-current-activity`/`current-package` | — | `mobile: getCurrent{Activity,Package}`. |
| `key-press` | `Keycode` | `mobile: pressKey {keycode}` (+`Params`). |
| `key-hide`/`key-shown` | — | `mobile: hideKeyboard` / `isKeyboardShown`. |
| `device-info`/`battery`/`time`/`notifications` | — | `mobile: deviceInfo`/`batteryInfo`/`getDeviceTime`/`openNotifications`. |
| `device-orientation`/`contexts` | — | `GET /orientation` / `GET /contexts`. |
| `device-set-orientation`/`set-clipboard` | `Params` | `POST /orientation` / `mobile: setClipboard`. |
| `device-context`/`get-clipboard` | — | `device-context`: empty `Params` → `GET /context`, set → `POST /context {name}`. |
| `execute` | `Expression` | `POST /execute/sync {script,args:[<RequestBody>]}`; `{element}` from `Selector`. |
| `raw` | `Method`+`Path` | arbitrary W3C call under `/session/<id>`; `{element}` from `Selector`. |

## Session-file format + location

`~/.cache/charly/appium/sessions/<image>[_<instance>].json` (XDG-cache; honours
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

- `charly/appium.go` — CLI subcommands + Run() methods. `session-create` uses
  `github.com/tebeka/selenium`'s `NewRemote` (which handles W3C
  `alwaysMatch` wrapping). All other ops use a small raw-HTTP W3C client
  (`w3cSession`) because the SDK can't attach to an existing session id.
- `charly/appium_session.go` — session-file load/save/delete + XDG path
  resolution.
- `charly/evalrun_charly_verbs.go` — `appiumMethods` allowlist + `runAppium`
  dispatcher.
- `charly/validate_eval.go` — `case "appium"` in `validateCharlyVerb`.

The host port is read from podman's `NetworkSettings.Ports` via
`InspectContainer` — same path as the adb verb.

## Library risks (documented)

`github.com/tebeka/selenium` last released v0.9.9 in 2022. The W3C
WebDriver protocol is stable so it works against Appium 3.x today, but
upstream activity is low. If it goes dormant we'll need to fork or
migrate to a newer library. The blast radius is small — only
`session-create` uses the SDK; the rest go through `w3cSession` (plain
HTTP), so an SDK swap is a single Run() method's worth of code.

`github.com/zach-klippenstein/goadb` (used by the sibling `/charly-eval:adb`)
has the same maintenance posture — pinned to a 2020 release. Same
mitigation if it stops working.

## Related skills

- `/charly-eval:adb` — sibling verb for low-level Android Debug Bridge
  control (install / shell / screencap / logcat).
- `/charly-eval:android` — the `kind: android` device + `apk:` package format +
  `target: android` deploy this UI automation runs against.
- `/charly-eval:eval` — the unified eval system and the Check struct that
  holds every verb discriminator + modifier.
- `/charly-tools:android-emulator` (when authored) — the image these verbs target.

## When to Use This Skill

**MUST be invoked** for any task involving `charly eval appium` commands or
`appium:` declarative checks. Invoke this skill BEFORE reading the
verb's Go source or reaching for `command: curl http://localhost:.../wd/hub/...`
workarounds.
