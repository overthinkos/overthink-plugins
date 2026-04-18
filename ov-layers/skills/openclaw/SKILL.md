---
name: openclaw
description: |
  OpenClaw AI gateway service on port 18789 via npm with persistent data.
  Use when working with OpenClaw, AI gateway configuration, or model routing.
---

# openclaw -- AI gateway service

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs`, `socat`, `supervisord` |
| Ports | 18789 |
| Port relay | 18789 (eth0 -> loopback) |
| Volumes | `data` -> `~/.openclaw` |
| Aliases | `openclaw` -> `openclaw` |
| Service | `openclaw` (supervisord) |
| Install files | `package.json` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `NODE_ENV` | `production` |

## Usage

```yaml
# image.yml
openclaw:
  layers:
    - openclaw
```

```bash
ov alias install openclaw    # install host 'openclaw' command
openclaw config              # uses the alias
```

## Used In Images

- `/ov-images:openclaw`
- `/ov-images:openclaw-sway-browser`
- `/ov-images:openclaw-ollama`
- `/ov-images:openclaw-ollama-sway-browser`

## Port Relay

The gateway binds to `127.0.0.1:18789` (loopback only). A socat relay forwards from the container's network interface to loopback. This avoids OpenClaw's `allowedOrigins` requirement for the Control UI — all connections appear as loopback to the gateway.

Same pattern as the `chrome` layer (CDP on port 9222).

## Related Layers

- `/ov-layers:nodejs` -- Node.js runtime dependency
- `/ov-layers:socat` -- port relay dependency (eth0 -> loopback)
- `/ov-layers:supervisord` -- process manager dependency
- `/ov-layers:ollama` -- optional local LLM backend

## When to Use This Skill

Use when the user asks about:

- OpenClaw gateway setup or configuration
- AI model routing or gateway
- Port 18789 service
- The `openclaw` host alias
- OpenClaw data volume
