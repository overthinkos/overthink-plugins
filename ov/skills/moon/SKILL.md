---
name: moon
description: |
  GameStream client protocol: ov moon commands for client-side pairing,
  app management, and session launch/quit with Sunshine servers.
  Use when working with Moonlight client operations.
---

# Moon - GameStream Client Protocol

## Overview

`ov moon` implements the GameStream HTTPS client protocol (ports 47984/47989) in pure Go. This is the same protocol that Moonlight clients use to pair with Sunshine, list apps, and launch/quit streaming sessions.

**`ov sun` = server management (Sunshine REST API) | `ov moon` = client protocol (GameStream HTTPS)**

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Server info | `ov moon status <image>` | GPU, state, running app (no pairing needed) |
| Pair | `ov moon pair <image> [pin] [--auto]` | Client-side pairing (--auto: single command) |
| Unpair | `ov moon unpair <image>` | Remove this client's pairing |
| List apps | `ov moon apps <image>` | List GameStream applications (requires pairing) |
| Launch app | `ov moon launch <image> <app>` | Start a streaming session |
| Quit app | `ov moon quit <image>` | Stop the running application |

## Architecture

```
CLI command -> resolveMoonContainer (engine + container name)
           -> resolveMoonHTTPAddress (port 47989) / resolveMoonHTTPSAddress (port 47984)
           -> ensureMoonCert (generate RSA-2048 client cert if needed)
           -> loadOrGenerateUniqueID (persistent 16-char hex client identity)
           -> GameStreamClient -> HTTP/HTTPS with XML responses
```

Uses two ports:
- **Port 47989 (HTTP):** Pairing, unpair, serverinfo (no TLS client cert required)
- **Port 47984 (HTTPS, mutual TLS):** App list, launch, resume, cancel (requires paired client cert)

## Client Certificate

`ov moon` generates and stores a self-signed RSA-2048 X.509 certificate per image:

**Storage:** `~/.config/ov/moonlight/<image>[-<instance>]/`
- `client.crt` — PEM certificate
- `client.key` — PEM private key
- `server.crt` — Server cert pinned after pairing
- `uniqueid` — 16-char hex client identity

## Commands

### Status
```bash
ov moon status sway-browser-sunshine
# Hostname:  my-host
# GPU:       NVIDIA GeForce RTX 4080 SUPER
# Version:   7.1.431.0
# Paired:    yes
# Running:   (none)
# State:     free
```

No pairing required — uses HTTP `/serverinfo` endpoint.

### Pair
```bash
ov moon pair sway-browser-sunshine --auto    # fully automated (recommended)
ov moon pair sway-browser-sunshine           # manual: generates PIN, waits for ov sun pair
ov moon pair sway-browser-sunshine 1234      # manual with specific PIN
```

The `--auto` flag handles everything in a single command: generates a PIN, starts the 5-phase handshake, and submits the PIN to Sunshine's REST API via a background goroutine. No manual step needed.

**5-phase cryptographic handshake:**
1. Exchange client/server certificates (blocks until PIN is submitted)
2. AES challenge-response using PIN-derived key
3. SHA-256 hash verification
4. RSA signature exchange
5. HTTPS mutual TLS verification

**Stale state handling:** Each pairing attempt automatically sends `GET /unpair` first to clear any leftover sessions from previous aborted attempts. This prevents EOF errors without needing to restart Sunshine.

**Manual pairing** — without `--auto`, combine with `ov sun pair`:
```bash
ov moon pair sway-browser-sunshine 1234 &   # starts handshake (blocks on phase 1)
sleep 3
ov sun pair sway-browser-sunshine 1234       # submits PIN server-side
```

### Unpair
```bash
ov moon unpair sway-browser-sunshine
```

### List Apps
```bash
ov moon apps sway-browser-sunshine
# 1   Desktop
# 2   Steam
```

Requires pairing (uses HTTPS with client cert).

### Launch
```bash
ov moon launch sway-browser-sunshine Desktop    # by name
ov moon launch sway-browser-sunshine 1          # by ID
```

Sends Sunshine's `/launch` API call to start a GameStream app. App name matching is case-insensitive. Generates random session encryption keys (`rikey`/`rikeyid`).

**Important:** This is a **control plane** command only. It tells Sunshine to start the app, but does NOT establish actual video/audio/input streaming. A real Moonlight client (the GUI application) handles the full streaming pipeline: RTSP session negotiation (port 48010), video decoding (UDP 47998), audio (UDP 48000), and input forwarding (UDP 47999). `ov moon launch` is useful for pre-starting an app before a Moonlight client connects.

### Quit
```bash
ov moon quit sway-browser-sunshine
```

Sends `/cancel` to Sunshine to stop the running app. Useful for cleaning up stale sessions.

## What `ov moon` Is and Isn't

**`ov moon` IS:**
- A pure Go implementation of the GameStream HTTPS control protocol
- An API client that manages pairing, app listing, session start/stop
- A tool for automating Sunshine setup without the Moonlight GUI

**`ov moon` is NOT:**
- A video/audio streaming client (no RTSP, no H.264/HEVC decoding)
- An input forwarder (no keyboard/mouse/gamepad streaming)
- A replacement for the Moonlight GUI/CLI application

For actual streaming with video and input, use the **Moonlight client** (AppImage in the `moonlight` layer, or desktop Moonlight app). `ov moon` complements Moonlight by handling pairing and session management programmatically.

## Desktop Interaction

For screenshots, clicks, and typing in Sunshine containers, use `ov wl` (which works via `podman exec` regardless of the remote access method):

```bash
ov wl screenshot sway-browser-sunshine          # grim
ov wl click sway-browser-sunshine 960 540       # wlrctl
ov wl type sway-browser-sunshine "hello"         # wtype
ov wl key sway-browser-sunshine Return           # wtype -k
```

## Testing Streaming End-to-End

`ov moon` handles the **control plane** (pairing, app launch/quit) but does NOT stream video/audio/input. To test actual streaming, use the Moonlight GUI client inside `sway-browser-vnc-moonlight`:

```bash
ov start sunshine-desktop-x11                   # Sunshine server (X11, recommended)
ov sun passwd sunshine-desktop-x11 --generate   # Set credentials
ov start sway-browser-vnc-moonlight             # Moonlight GUI client
# VNC into sway-browser-vnc-moonlight, press Alt+M, pair via Moonlight GUI
```

The Moonlight GUI performs its own independent pairing (separate from `ov moon pair` on the host). Only the GUI handles the full streaming pipeline: RTSP negotiation, H.264/HEVC decoding, audio, and input forwarding.

## Cross-References

- `/ov:sun` — Sunshine server management (credentials, config, REST API)
- `/ov:wl` — Desktop interaction (screenshot, click, type via Wayland tools)
- `/ov:cdp` — Chrome DevTools Protocol (browser automation)
- `/ov-layers:sunshine-x11` — Sunshine with X11 capture (recommended)
- `/ov-layers:sunshine` — Sunshine with Sway/wlroots capture
- `/ov-images:sunshine-desktop-x11` — X11 Sunshine server image (recommended)
- `/ov-images:sway-browser-vnc-moonlight` — Moonlight GUI client for testing

## When to Use This Skill

**MUST be invoked** when the task involves GameStream client operations, Moonlight pairing, app launching/quitting, or `ov moon` commands. Invoke this skill BEFORE reading source code or launching Explore agents.

**Workflow position:** Client-side streaming protocol. Use `ov sun` for server setup, `ov moon` for client pairing, `ov wl` for desktop interaction.
