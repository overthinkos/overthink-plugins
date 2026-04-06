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

```bash
ov build selkies-desktop-hermes-jupyter
ov config selkies-desktop-hermes-jupyter -e OPENROUTER_API_KEY=sk-xxx
ov start selkies-desktop-hermes-jupyter
# Open https://localhost:3000  (Selkies desktop)
# Open http://localhost:8888   (JupyterLab)
```

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

## Related Images

- `/ov-images:selkies-desktop-hermes` — same image without JupyterLab
- `/ov-images:selkies-desktop` — desktop only (no Hermes, no Jupyter)
- `/ov-images:jupyter-colab` — lightweight JupyterLab only (no desktop, no Hermes)

## When to Use This Skill

**MUST be invoked** when the task involves the selkies-desktop-hermes-jupyter image, combining Selkies desktop + Hermes + JupyterLab, or the MCP server alongside the streaming desktop.
