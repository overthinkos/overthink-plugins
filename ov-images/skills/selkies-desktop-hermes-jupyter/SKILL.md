---
name: selkies-desktop-hermes-jupyter
description: |
  Combined Selkies streaming desktop + Hermes AI agent + JupyterLab with CRDT collaboration and MCP server.
  Use when working with the selkies-desktop-hermes-jupyter image.
---

# selkies-desktop-hermes-jupyter

Browser-accessible Wayland desktop (Selkies/pixelflux) + Hermes AI agent + JupyterLab with real-time CRDT collaboration and MCP server. Extends `selkies-desktop-hermes` with the `jupyter-colab` layer.

## Image Properties

| Property | Value |
|----------|-------|
| Base | selkies-desktop-hermes |
| Layers | jupyter-colab (on top of base) |
| Platforms | linux/amd64 |
| Registry | ghcr.io/overthinkos |

## Definition

```yaml
selkies-desktop-hermes-jupyter:
  base: selkies-desktop-hermes
  layers:
    - jupyter-colab
  ports:
    - "3000:3000"
    - "8888:8888"
    - "9222:9222"
  platforms:
    - linux/amd64
```

## Base Stack (from selkies-desktop-hermes)

All layers from `selkies-desktop-hermes` are inherited:
- `fedora-nonfree` (Fedora 43 + RPM Fusion)
- `selkies-desktop` metalayer (pipewire + chrome + labwc + waybar + selkies + wl-tools + ...)
- `hermes` (Hermes AI agent, nodejs, ffmpeg, supervisord)
- `claude-code`, `codex`, `gemini` (AI CLI tools)
- `agent-forwarding`, `dbus`, `ov`

## Added Layer

`jupyter-colab` — adds JupyterLab 4.x + jupyter-collaboration (Y-CRDT) + data science stack + MCP server (13 tools). Pixi env overlays on top of the hermes pixi env. Packages have minimal overlap (JupyterLab/data science vs. AI/agent), so both environments coexist cleanly via Docker overlay FS.

## Ports

| Port | Service |
|------|---------|
| 3000 | Selkies web UI (Traefik HTTPS) |
| 8888 | JupyterLab + MCP server (`/mcp`) |
| 9222 | Chrome DevTools Protocol (CDP) |

## Volumes

| Volume | Path | From |
|--------|------|------|
| `chrome-data` | `~/.chrome-debug` | selkies-desktop |
| `selkies-config` | `~/.config/selkies` | selkies-desktop |
| `data` | `/opt/data` | hermes |
| `workspace` | `~/workspace` | jupyter-colab |

## Services

| Service | Description |
|---------|-------------|
| `dbus` | D-Bus session bus |
| `pipewire` | Audio (Selkies streaming + Hermes voice) |
| `selkies` | Wayland capture + H.264/Opus streaming |
| `labwc` | Wayland compositor |
| `swaync` | Notification daemon |
| `waybar` | Status bar |
| `traefik` | HTTPS reverse proxy |
| `selkies-fileserver` | Static file server |
| `relay-9222` | CDP port relay |
| `hermes` | Hermes AI agent (autostart) |
| `hermes-whatsapp` | WhatsApp bridge (manual) |
| `jupyter-colab` | JupyterLab server (autostart) |

## Quick Start

The hermes entrypoint auto-configures the LLM provider based on which env var is set (priority: `OLLAMA_HOST` > `OLLAMA_API_KEY` > `OPENROUTER_API_KEY`):

```bash
ov build selkies-desktop-hermes-jupyter

# Option 1: Ollama Cloud (no local GPU needed)
ov config selkies-desktop-hermes-jupyter -e OLLAMA_API_KEY=your-key
# Auto-configures: kimi-k2.5:cloud via https://ollama.com/v1

# Option 2: OpenRouter
ov config selkies-desktop-hermes-jupyter -e OPENROUTER_API_KEY=sk-or-xxx
# Auto-configures: qwen/qwen3.6-plus:free via OpenRouter

# Option 3: Local Ollama sidecar (OLLAMA_HOST auto-injected by ov config ollama --update-all)
ov config selkies-desktop-hermes-jupyter
# Auto-configures: qwen2.5-coder:32b via local Ollama

# Override default model with any provider:
ov config selkies-desktop-hermes-jupyter -e OLLAMA_API_KEY=your-key -e HERMES_MODEL=llama3.3:cloud

ov start selkies-desktop-hermes-jupyter
# Open https://localhost:3000  (Selkies desktop)
# Open http://localhost:8888   (JupyterLab)
```

## Tailnet Deployment

Expose ports 3000 and 8888 to all hosts in your Tailscale network:

```bash
ov deploy import <<EOF
images:
  selkies-desktop-hermes-jupyter:
    tunnel:
      provider: tailscale
      private: [3000, 8888]
EOF

ov config selkies-desktop-hermes-jupyter
ov start selkies-desktop-hermes-jupyter

# Access from any tailnet host:
#   https://<hostname>:3000  (Selkies desktop, https+insecure backend)
#   https://<hostname>:8888  (JupyterLab, http backend)
```

Port 3000 uses `https+insecure` backend scheme (Traefik self-signed cert). Port 8888 uses plain `http`. Port 9222 (CDP) is intentionally not tunneled — it allows arbitrary JS execution.

Requires: `sudo systemctl enable --now tailscaled && tailscale up && sudo tailscale set --operator=$USER`

## MCP Server

The JupyterLab MCP server is at `http://localhost:8888/mcp` (Streamable HTTP, MCP spec 2025-11-25) with 13 tools for programmatic notebook access.

```bash
claude mcp add --transport http --scope project jupyter-colab http://localhost:8888/mcp
```

## Verification

```bash
ov status selkies-desktop-hermes-jupyter              # container running
ov service status selkies-desktop-hermes-jupyter      # all 12 services RUNNING
curl -k https://localhost:3000                        # 200 OK (Selkies)
curl -o /dev/null -w '%{http_code}' http://localhost:8888  # 302 → /lab
ov wl screenshot selkies-desktop-hermes-jupyter t.png # desktop visible
ov cdp status selkies-desktop-hermes-jupyter          # CDP ok
ov shell selkies-desktop-hermes-jupyter -c "hermes --version"  # v0.7.0
ov shell selkies-desktop-hermes-jupyter -c "jupyter lab --version"  # 4.x
```

## MCP Auto-Discovery

Hermes auto-discovers the jupyter-colab MCP server at first start. In this combined image, both services run in the same container, so the URL resolves to `http://localhost:8888/mcp` (pod-aware).

**Verify:**
```bash
ov shell selkies-desktop-hermes-jupyter -c "hermes mcp test jupyter-colab"
# Expects 13 tools, ~742ms connection
```

**Use:**
```bash
hermes chat -q "Use the list_notebooks MCP tool to list notebooks"
```
Or in interactive chat: `/reload-mcp` to refresh MCP server connections.

No manual MCP registration needed -- configured automatically by the entrypoint from `OV_MCP_SERVERS`.

## Related Images

- `/ov-images:selkies-desktop-hermes` — same image without JupyterLab
- `/ov-images:selkies-desktop` — desktop only (no Hermes, no Jupyter)
- `/ov-images:jupyter-colab` — lightweight JupyterLab only (no desktop, no Hermes)

## When to Use This Skill

**MUST be invoked** when the task involves the selkies-desktop-hermes-jupyter image, combining Selkies desktop + Hermes + JupyterLab, or the MCP server alongside the streaming desktop.
