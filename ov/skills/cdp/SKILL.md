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
| Click | `ov cdp click <image> <tab-id> <selector>` | Click element by CSS selector |
| Type | `ov cdp type <image> <tab-id> <selector> <text>` | Type into input field |
| Eval JS | `ov cdp eval <image> <tab-id> <expression>` | Evaluate JavaScript |
| Wait | `ov cdp wait <image> <tab-id> <selector>` | Wait for element (--timeout 30s) |
| Raw CDP | `ov cdp raw <image> <tab-id> <method> [json]` | Send raw CDP command |

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

Click uses CDP: `Runtime.evaluate` to find element, `DOM.getBoxModel` for coordinates, `Input.dispatchMouseEvent` for click. Type uses `DOM.focus` and `Input.dispatchKeyEvent` for each character.

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

## browser-open Script and BROWSER Env

Images with Chrome include a `browser-open` script and set `BROWSER=browser-open` in the environment. When CLI tools inside the container call `xdg-open` or use the `$BROWSER` variable to open a URL, it routes through CDP to open the URL in the running Chrome instance.

## OAuth Automation Example

Complete flow for deploying openclaw with Codex OAuth:

```bash
# 1. Build and deploy with tailscale serve on all ports
ov build openclaw-sway-browser
ov enable openclaw-sway-browser
# Quadlet generates: tailscale serve --https=18789, --tcp=5900, --https=9222

# 2. Start OAuth process inside the running container
ov shell openclaw-sway-browser --tty -c \
  "openclaw models auth login --provider openai-codex --set-default" > /tmp/oauth.log &

# 3. Extract OAuth URL, open in container's Chrome via CDP
OAUTH_URL=$(grep -oP 'https://auth\.openai\.com\S+' /tmp/oauth.log)
ov cdp open openclaw-sway-browser "$OAUTH_URL"

# 4. Wait for login page, click "Continue with Google"
TAB=$(ov cdp list openclaw-sway-browser | grep openai | awk '{print $1}')
ov cdp wait openclaw-sway-browser $TAB 'button[value="google"]'
ov cdp click openclaw-sway-browser $TAB 'button[value="google"]'

# 5. Wait for consent page, click "Continue"
ov cdp wait openclaw-sway-browser $TAB 'button[type="submit"]'
ov cdp click openclaw-sway-browser $TAB 'button[type="submit"]'

# 6. OAuth callback hits localhost:1455 inside container
# Tokens saved to ~/.openclaw volume
```

Key enablers:
- `--tty` provides PTY for interactive CLI commands from automation
- `port_relay` makes Chrome DevTools accessible from host through podman bridge networking
- `browser-open` + `BROWSER` env lets CLI tools open URLs in the running Chrome via CDP
- `tailscale serve --tcp=5900` exposes VNC for manual inspection if needed
- `shm_size: 1g` prevents Chrome from crashing due to /dev/shm exhaustion

Source: `ov/cdp.go`, `ov/browser_cdp.go`.

## Cross-References

- `/ov:sway` -- Sway compositor control (window management, workspaces)
- `/ov:vnc` -- VNC desktop automation (same container, pixel-level interaction)
- `/ov:shell` -- Running commands in containers (`--tty` for OAuth flows)
- `/ov:service` -- Starting containers (`ov start`, `ov enable`)
- `/ov:layer` -- `port_relay` field and Chrome layer configuration

## When to Use This Skill

**MUST be invoked** when the task involves Chrome DevTools Protocol, ov cdp commands, browser automation, clicking elements, taking screenshots, or OAuth flows inside containers. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Desktop automation. Use after a desktop container is running. Preferred over VNC for structured interaction. See also `/ov:vnc` (pixel), `/ov:sway` (window).
