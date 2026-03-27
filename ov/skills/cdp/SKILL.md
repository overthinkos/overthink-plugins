---
name: cdp
description: |
  MUST be invoked before any work involving: Chrome DevTools Protocol, ov cdp commands, browser automation, clicking elements, taking screenshots, or OAuth flows inside containers.
---

# CDP - Chrome DevTools Protocol

## Overview

`ov cdp` commands connect to Chrome DevTools Protocol (CDP) on port 9222 inside running containers. Provides HTTP API operations (open, list, close tabs) and WebSocket CDP operations (click, type, eval, wait, text, html, screenshot) for headless browser automation.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Open URL | `ov cdp open <image> <url>` | Open URL in new Chrome tab |
| List tabs | `ov cdp list <image>` | List all open tabs (id, title, url) |
| Close tab | `ov cdp close <image> <tab-id>` | Close a tab by ID |
| Get text | `ov cdp text <image> <tab-id>` | Get page text content |
| Get HTML | `ov cdp html <image> <tab-id>` | Get page HTML source |
| Get URL | `ov cdp url <image> <tab-id>` | Get page title and URL |
| Screenshot | `ov cdp screenshot <image> <tab-id> [file]` | Capture PNG screenshot |
| Click | `ov cdp click <image> <tab-id> <selector> [--vnc]` | Click element by CSS selector |
| Coords | `ov cdp coords <image> <tab-id> <selector>` | Show element coords in viewport + desktop |
| Type | `ov cdp type <image> <tab-id> <selector> <text>` | Type into input field |
| Eval JS | `ov cdp eval <image> <tab-id> <expression>` | Evaluate JavaScript |
| Wait | `ov cdp wait <image> <tab-id> <selector>` | Wait for element (--timeout 30s) |
| Raw CDP | `ov cdp raw <image> <tab-id> <method> [json]` | Send raw CDP command |
| Status | `ov cdp status <image>` | Check CDP availability, show port and tab count |

All commands accept `-i INSTANCE` for multi-instance support.

## Architecture

1. Resolves the container name from image + instance (`ov-<image>[-<instance>]`)
2. Discovers the mapped port 9222 via `podman port` / `docker port`
3. **HTTP API** (`/json/list`, `/json/new?url=`, `/json/close/<id>`) for list, open, and close
4. **CDP WebSocket** for interactive operations (click, type, eval, wait, text, html, screenshot, cdp)

## Requirements

- A Chrome layer with `port_relay: [9222]` in layer.yml
- Chrome launched with `--remote-allow-origins='*'` and `--remote-debugging-port=9222`
- Container must be running (`ov start` or `ov enable`)

The `port_relay` is essential because Chrome 146+ binds DevTools only to 127.0.0.1. The socat relay forwards eth0:9222 to localhost:9222, making it accessible through container port mappings.

## Commands

### Open a URL

```bash
ov cdp open my-app "https://example.com"
```

Uses HTTP API: `PUT /json/new?url=<encoded-url>`. Returns the new tab ID.

### List Tabs

```bash
ov cdp list my-app
# ID                                    TITLE                URL
# 7F8A3B2C...                          Example Domain       https://example.com/
```

Uses HTTP API: `GET /json/list`.

### Close a Tab

```bash
ov cdp close my-app 7F8A3B2C...
```

Uses HTTP API: `GET /json/close/<id>`.

### Get Page Content

```bash
ov cdp text my-app $TAB      # Plain text
ov cdp html my-app $TAB      # HTML source
ov cdp url my-app $TAB       # Title and URL
```

Uses CDP WebSocket: `Runtime.evaluate` with `document.body.innerText` / `document.documentElement.outerHTML`, `Target.getTargetInfo`.

### Screenshot

```bash
ov cdp screenshot my-app $TAB              # Prints base64 to stdout
ov cdp screenshot my-app $TAB page.png     # Saves to file
```

Uses CDP: `Page.captureScreenshot`.

### Click and Type

```bash
ov cdp click my-app $TAB 'button[type="submit"]'
ov cdp type my-app $TAB 'input[name="email"]' "user@example.com"
```

Click uses CDP: `Runtime.evaluate` with `deepQuery()` to find element (piercing shadow DOM), `scrollIntoViewIfNeeded()` + `getBoundingClientRect()` for coordinates, `Input.dispatchMouseEvent` for click. Type uses `deepQuery()` + `scrollIntoViewIfNeeded()` + `focus()` to select the element, then `Input.dispatchKeyEvent` for each character (keyDown, char, keyUp — matching Puppeteer behavior).

**Shadow DOM support:** All selector-based commands (click, type, wait) automatically pierce shadow DOM boundaries via recursive `deepQuery()`. This means selectors work on Chrome's internal pages (`chrome://settings/*`), Polymer/Lit web components, and any page using Web Components with shadow DOM. Hidden/zero-sized elements are skipped — only visible matches are returned.

**Note on Chrome internal dialogs:** Some Chrome UI elements (e.g., the "Turn on sync" confirmation dialog) are rendered as native browser chrome, invisible to CDP. Use VNC keyboard (`ov vnc key ... Tab`, `ov vnc key ... Return`) to interact with these dialogs. VNC screenshots (`ov vnc screenshot`) show the full desktop including these dialogs, while CDP screenshots only show the page viewport.

### Coordinate Systems

CDP coordinates are **viewport-relative** (relative to Chrome's content area). VNC coordinates are **desktop-absolute** (the full Wayland framebuffer). The offset between them includes Chrome's window position on the desktop plus Chrome's UI chrome (title bar, tab bar, address bar — typically ~107px).

**`ov cdp coords`** — Shows an element's coordinates in both systems:

```bash
ov cdp coords my-app $TAB '#sync-button'
# Element:  #sync-button (108x36)
# Viewport: x=1166 y=310  center=(1220, 328)
# Desktop:  x=1166 y=421  center=(1220, 439)  (via window.screenX/screenY, chromeHeight=107)
# Sway:     window at (4, 4) size 1912x1032  (app_id=google-chrome)
```

**`ov cdp click --vnc`** — Finds element via CDP selector, delivers click via VNC:

```bash
ov cdp click my-app $TAB '#sync-button' --vnc
# Clicked element at viewport (1220, 328) → desktop (1220, 439) via VNC
```

**`ov vnc click --from-cdp`** — Translates viewport coords to desktop coords:

```bash
ov vnc click my-app 1220 328 --from-cdp $TAB
# Translated viewport (1220, 328) → desktop (1220, 439) via CDP tab ...
```

### Evaluate JavaScript

```bash
ov cdp eval my-app $TAB 'document.title'
ov cdp eval my-app $TAB 'JSON.stringify(localStorage)'
```

Uses CDP: `Runtime.evaluate`. Returns the result value.

### Wait for Element

```bash
ov cdp wait my-app $TAB 'h1'                    # Default 30s timeout
ov cdp wait my-app $TAB '.loaded' --timeout 60s  # Custom timeout
```

Polls with CDP until the CSS selector matches an element.

### Raw CDP Command

```bash
ov cdp raw my-app $TAB 'Page.navigate' '{"url":"https://example.com"}'
ov cdp raw my-app $TAB 'Runtime.evaluate' '{"expression":"1+1"}'
```

Sends arbitrary CDP method with optional JSON params. Returns raw CDP response.

## CDP Connection Diagnostics

When `ov cdp` commands fail to connect, the `diagnoseCDP()` function runs automatically and provides targeted hints:

1. **Chrome process check**: Is Chrome running inside the container? (`pgrep chrome`)
2. **Relay status**: Is the port relay (socat) forwarding 9222? (`supervisorctl status relay-9222`)
3. **Port binding**: Is Chrome listening on 127.0.0.1:9222? (`ss -tlnp`)

Hints direct users to `ov wl sway exec <image> chrome-wrapper` for manual Chrome restart (not `ov shell` with bare `swaymsg`, which may lack the correct `SWAYSOCK` path).

## browser-open Script and BROWSER Env

Images with Chrome include a `browser-open` script and set `BROWSER=browser-open` in the environment. When CLI tools inside the container call `xdg-open` or use the `$BROWSER` variable to open a URL, it routes through CDP to open the URL in the running Chrome instance.

## OAuth Automation Example

Complete flow for deploying openclaw with Codex OAuth. **All browser interactions must be VNC-visible** — use `--vnc` flag on `ov cdp click`.

**Critical:** The `openclaw models auth login` TUI requires a real terminal. Do NOT pipe through `tee` or redirect stdout. Use **`ov tmux`** (see `/ov:tmux`).

```bash
IMG=openclaw-sway-browser  # or openclaw-ollama-sway-browser

# 1. Prerequisites: Chrome signed into Google with sync enabled
#    See /ov-images:openclaw-ollama-sway-browser for full Chrome sign-in procedure

# 2. Start OAuth in a tmux session (real terminal)
ov tmux run $IMG -s oauth "openclaw models auth login --provider openai-codex --set-default"

# 3. Read OAuth URL from tmux output
sleep 5
OAUTH_URL=$(ov tmux capture $IMG -s oauth | grep -o 'https://auth.openai.com/[^ ]*')
ov cdp open $IMG "$OAUTH_URL"

# 4. Click "Continue with Google" (VNC-visible)
sleep 5
TAB=$(ov cdp list $IMG | grep -i "openai\|auth" | head -1 | awk '{print $1}')
ov cdp click $IMG $TAB 'button._buttonStyleFix_wvuha_65' --vnc

# 5. Click "Continue" on Codex consent page (VNC-visible)
sleep 5
ov cdp click $IMG $TAB 'button._primary_3rdp0_107' --vnc

# 6. Verify token exchange completed
sleep 10
ov tmux capture $IMG -s oauth
# Expected: "OpenAI OAuth complete", "Default model set to openai-codex/gpt-5.4"

# 7. Restart gateway
ov shell $IMG -c "supervisorctl restart openclaw"
ov shell $IMG -c "openclaw models status"
```

**Tested selectors** (OpenAI auth page as of 2026-03-21):
- "Continue with Google": `button._buttonStyleFix_wvuha_65` (first matching social button)
- "Continue" (consent): `button._primary_3rdp0_107` (black primary button)
- These are CSS class selectors specific to OpenAI's auth UI — may change over time

Key enablers:
- `ov tmux run` provides a real terminal for the TUI to complete the token exchange (see `/ov:tmux`)
- `ov cdp click --vnc` finds elements via CDP, clicks via VNC (visible to user)
- `port_relay` makes Chrome DevTools accessible from host through podman bridge networking
- `shm_size: 1g` prevents Chrome from crashing due to /dev/shm exhaustion
- Callback at `localhost:1455` is container-internal (no port mapping needed)

**Stale port 1455:** If a previous OAuth attempt left port 1455 occupied: `ov shell $IMG -c 'kill -9 $(ss -tlnp sport = :1455 | grep -oP "pid=\K\d+")'`

Source: `ov/cdp.go`, `ov/vnc.go`.

## Google Sign-In Automation

Sign into a Google account inside a running container. Requires `GMAIL_USER` and `GMAIL_PASSWORD` environment variables (set in `.env` or passed via `-e`).

**App Passwords required:** Google accounts with 2FA (now mandatory for most accounts) require a 16-character [App Password](https://myaccount.google.com/apppasswords). App Passwords bypass all verification challenges and 2FA prompts — use them by default for automated sign-in.

**Fresh profile prerequisite:** A fresh `chrome-data` volume triggers Chrome's first-run flow. Use `ov remove <image> --purge` before `ov enable` to ensure a clean start. Just rebuilding the image does not reset named volumes.

### Step 0: Dismiss Chrome First-Run Dialog

On a fresh profile, Chrome opens a first-run dialog ("Make Google Chrome the default browser") as a **separate window** that CDP cannot see (no debuggable tabs). It tiles alongside any CDP-opened tabs in sway, breaking coordinate translation.

```bash
# Focus the first-run dialog and dismiss it
ov wl sway msg my-app 'focus left'     # first-run dialog is typically the left window
ov vnc key my-app Return            # press OK
```

After dismissal, Chrome shows `chrome://intro/` — "Sign in to Chrome" with shadow DOM buttons.

### Step 1: Click "Sign in" on chrome://intro

`chrome://` pages block CDP mouse events and JS `.click()`. Use `--vnc` click (CDP selector targeting + VNC pointer delivery):

```bash
TAB=$(ov cdp list my-app | grep intro | head -1 | awk '{print $1}')
ov cdp click my-app $TAB '#acceptSignInButton' --vnc
```

Shadow DOM path: `intro-app` > `sign-in-promo` > `#acceptSignInButton`. The `--vnc` flag uses `deepQuery` to find the element, translates viewport coords to desktop coords via `window.screenX/screenY`, and delivers the click through VNC.

This opens a new tab with the Google sign-in page. Capture the new tab ID:

```bash
sleep 3
TAB=$(ov cdp list my-app | grep -i "sign in" | head -1 | awk '{print $1}')
```

**Note:** The tab ID survives Google's same-tab navigations (email → password → result).

### Step 2: Enter Email (--vnc click + VNC type)

```bash
ov cdp wait my-app $TAB '#identifierId' --timeout 30s
ov cdp click my-app $TAB '#identifierId' --vnc    # focus field via VNC pointer
sleep 0.5                                          # let compositor process focus
ov vnc type my-app "$GMAIL_USER"                   # real keysym events
```

Use `ov cdp coords my-app $TAB '#identifierId'` to inspect element position in all three coordinate systems (viewport, desktop via CDP, desktop via sway) for debugging.

### Step 3: Submit Email

```bash
ov cdp click my-app $TAB '#identifierNext' --vnc
sleep 5                                            # page transition
ov cdp url my-app $TAB                             # expect /challenge/pwd
ov cdp screenshot my-app $TAB step3.png            # verification checkpoint
```

### Step 4: Enter Password

```bash
ov cdp wait my-app $TAB 'input[type="password"]' --timeout 15s
ov cdp click my-app $TAB 'input[type="password"]' --vnc
sleep 0.5
ov vnc type my-app "$GMAIL_PASSWORD"
```

### Step 5: Submit Password

```bash
ov cdp click my-app $TAB '#passwordNext' --vnc
sleep 7                                            # backend verification
ov cdp screenshot my-app $TAB step5.png
ov vnc screenshot my-app step5-desktop.png         # catches native dialogs
```

### Step 6: Enable Sync (chrome://sync-confirmation)

After successful sign-in, Chrome navigates to `chrome://sync-confirmation/` — a `chrome://` page (NOT a native dialog). CDP can see it but `--vnc` click is required:

```bash
TAB=$(ov cdp list my-app | grep -i sync-confirmation | head -1 | awk '{print $1}')
# If no sync-confirmation tab, it may already be on the current tab:
# TAB stays the same from step 5
ov cdp click my-app $TAB '#confirmButton' --vnc    # "Yes, I'm in"
```

Shadow DOM path: `sync-confirmation-app` > `#confirmButton`. Other buttons: `#notNowButton` ("No thanks"), `#settingsButton` ("Settings").

### Handling Challenges

**2FA/CAPTCHA:** Take a VNC screenshot (`ov vnc screenshot my-app challenge.png`) and complete manually via a VNC client. App Passwords bypass most challenges.

**Search engine choice:** May appear as a new tab. Handle via CDP eval if present:
```bash
STAB=$(ov cdp list my-app | grep search-engine | head -1 | awk '{print $1}')
# If STAB is non-empty, select Google via shadow DOM eval
```

### Sign-In Persistence

Cookies and sync state are stored in the `chrome-data` volume (`~/.chrome-debug`), persisting across container restarts. Use `ov remove <image> --purge` to clear for a fresh start.

### Coordinate Translation for Sign-In

The `--vnc` flag on `ov cdp click` is essential for the sign-in flow:
- **`chrome://` pages** (intro, sync-confirmation): CDP mouse events and JS `.click()` are blocked. `--vnc` is the only way to click.
- **Google sign-in pages**: `--vnc` delivers real pointer events that bypass anti-automation detection.
- **Coordinate math**: viewport center + `window.screenX` + `window.screenY` + `chromeHeight` = desktop coords. On popup windows (no toolbar), `chromeHeight=0`.

Use `ov cdp coords my-app $TAB '<selector>'` to debug coordinate translation. It shows element position in viewport, desktop (via CDP), and desktop (via sway) systems.

## Cross-References

- `/ov:wl` (sway subgroup) -- Sway compositor control (window management, workspaces)
- `/ov:vnc` -- VNC desktop automation (same container, pixel-level interaction)
- `/ov:shell` -- Running commands in containers (`--tty` for OAuth flows)
- `/ov:service` -- Starting containers (`ov start`, `ov enable`)
- `/ov:layer` -- `port_relay` field and Chrome layer configuration

## When to Use This Skill

**MUST be invoked** when the task involves Chrome DevTools Protocol, ov cdp commands, browser automation, clicking elements, taking screenshots, or OAuth flows inside containers. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Desktop automation. Use after a desktop container is running. Preferred over VNC for structured interaction. See also `/ov:vnc` (pixel), `/ov:wl` (sway subgroup) (window).
