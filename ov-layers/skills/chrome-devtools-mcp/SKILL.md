---
name: chrome-devtools-mcp
description: |
  Chrome DevTools MCP server via mcp-proxy (Streamable HTTP on port 9224).
  Use when working with the chrome-devtools-mcp layer, MCP-based browser automation,
  or the mcp-proxy stdio-to-HTTP bridge pattern.
---

# chrome-devtools-mcp -- Chrome DevTools MCP Server

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs`, `supervisord` |
| Ports | 9224 (Streamable HTTP MCP endpoint) |
| Services | `chrome-devtools-mcp` (supervisord, autostart) |
| Install files | `layer.yml`, `pixi.toml`, `package.json` |

## MCP Provides

| Name | URL Template | Transport |
|------|-------------|-----------|
| `chrome-devtools` | `http://{{.ContainerName}}:9224/mcp` | http |

Pod-aware: same-container consumers receive `http://localhost:9224/mcp`, cross-container consumers receive `http://ov-<image>:9224/mcp`. Consumed by hermes (`mcp_accepts: chrome-devtools`) for MCP-based browser inspection and automation.

## Architecture

```
MCP Client (hermes, Claude Code)
    ↓ Streamable HTTP (POST /mcp)
mcp-proxy (Python, pixi)  — 0.0.0.0:9224
    ↓ stdio
chrome-devtools-mcp (Node.js, npm)
    ↓ Chrome DevTools Protocol
cdp-proxy (port 9222) → Chrome (port 9223)
```

- **mcp-proxy** (Python, installed via pixi as PyPI dependency) bridges stdio-based chrome-devtools-mcp to Streamable HTTP on port 9224
- **chrome-devtools-mcp** (Node.js, installed via npm `package.json`) connects to Chrome at `http://127.0.0.1:9222` via the cdp-proxy
- Runs as a supervisord service with `autorestart=true` (handles startup ordering — retries until Chrome is ready)
- This is a dual-builder layer: pixi (mcp-proxy) + npm (chrome-devtools-mcp)

## Available Tools (29)

| Category | Tools |
|----------|-------|
| Input | `click`, `drag`, `fill`, `fill_form`, `hover`, `type_text`, `press_key`, `upload_file`, `handle_dialog` |
| Navigation | `navigate_page`, `list_pages`, `select_page`, `new_page`, `close_page`, `wait_for` |
| Inspection | `take_screenshot`, `take_snapshot`, `evaluate_script`, `get_console_message`, `list_console_messages` |
| Network | `list_network_requests`, `get_network_request` |
| Performance | `performance_start_trace`, `performance_stop_trace`, `performance_analyze_insight`, `take_memory_snapshot`, `lighthouse_audit` |
| Emulation | `emulate`, `resize_page` |

## Packages

- **pixi (PyPI):** `mcp-proxy`
- **npm:** `chrome-devtools-mcp`

## Key Insight: Hermes Reconfiguration

Hermes `config.yaml` is guarded by a `# ov:auto-configured` sentinel — it only generates on first start, never overwrites. When chrome-devtools-mcp is added to a running deployment, hermes won't automatically pick it up.

**Fix:** Delete config.yaml and restart hermes:
```bash
ov shell <image> -c "rm /opt/data/config.yaml"
ov service restart <image> hermes
```

After restart, `hermes mcp list` should show `chrome-devtools` as enabled with all 29 tools.

## Automatically Included Via

This layer is auto-included by the `chrome` base layer via `layers: [chrome-devtools-mcp]`. Any image that includes `chrome`, `chrome-sway`, `chrome-niri`, `chrome-mutter`, `chrome-kwin`, `chrome-x11`, or any desktop metalayer gets this automatically — zero explicit configuration needed.

## Used In Layers

- `/ov-layers:chrome` — includes via `layers: [chrome-devtools-mcp]`

## Used In Images (via chrome dependency chain)

- `/ov-images:selkies-desktop`
- `/ov-images:selkies-desktop-nvidia`
- `/ov-images:sway-browser-vnc`
- `/ov-images:openclaw-sway-browser`
- `/ov-images:openclaw-ollama-sway-browser`
- All other images containing a chrome variant

## Related Skills

- `/ov-layers:chrome` — parent layer, provides Chrome + CDP on port 9222
- `/ov-layers:jupyter-mcp` — analogous MCP server pattern (Tier 1, different domain)
- `/ov-layers:hermes` — consumes via `mcp_accepts: chrome-devtools`
- `/ov-layers:mcporter` — MCP server CLI (npm-based, similar npm install pattern)
- `/ov:cdp` — direct Chrome DevTools Protocol commands (lower-level than MCP)
- `/ov:mcp` — test-side client for this layer's MCP endpoint (`ov eval mcp ping`, `list-tools`, etc.). The layer ships 2 deploy-scope `mcp:` declarative checks (`mcp-chrome-devtools-ping`, `mcp-chrome-devtools-list-tools` asserting `navigate_page` / `take_screenshot`). **Port-publishing gotcha**: when this layer is added to an image that already has a `deploy.yml` `ports:` override (e.g. `sway-browser-vnc`), port 9224 may not be published until the override is updated. `ov eval mcp` surfaces this with the exact `ports: [9224:9224]` remediation message — see `/ov:mcp` for the full fix.

## When to Use This Skill

Use when the user asks about:

- The chrome-devtools-mcp layer or its MCP server
- MCP-based browser automation via Chrome DevTools (vs direct CDP)
- The mcp-proxy stdio-to-HTTP bridge pattern
- Port 9224 or the chrome-devtools MCP endpoint
- How Chrome DevTools MCP integrates with hermes
- Why hermes doesn't see a newly added MCP server (sentinel issue)

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
