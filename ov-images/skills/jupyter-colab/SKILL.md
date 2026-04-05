---
name: jupyter-colab
description: |
  Lightweight JupyterLab with real-time collaboration on port 8888. No GPU required.
  Based on fedora (not nvidia), supports both amd64 and arm64.
  MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter-colab image.
---

# jupyter-colab

Lightweight JupyterLab with real-time collaboration via jupyter-collaboration (Y-CRDT).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, jupyter-colab, dbus, ov |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 8888 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (Fedora 43 base — no GPU)
2. `pixi` → `python` → `supervisord` (transitive)
3. `jupyter-colab` — JupyterLab + jupyter-collaboration + data science
4. `agent-forwarding` — SSH/GPG agent forwarding
5. `dbus` — D-Bus session bus
6. `ov` — ov CLI binary

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8888 | JupyterLab | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| notebooks | ~/notebooks | Persistent notebook storage |

## Quick Start

```bash
ov build jupyter-colab
ov config jupyter-colab
ov start jupyter-colab
# Open http://localhost:8888
```

## Key Layers

- `/ov-layers:jupyter-colab` — JupyterLab + collaboration + data science
- `/ov-layers:agent-forwarding` — SSH/GPG forwarding

## Related Images

- `/ov-images:jupyter` — GPU-accelerated variant (nvidia base, ML libraries, amd64 only)
- `/ov-images:fedora` — parent base image

## Verification

After `ov start`:

```bash
# Container and services running
ov status jupyter-colab
ov service status jupyter-colab

# JupyterLab responds
curl -s -o /dev/null -w '%{http_code}' http://localhost:8888    # 200

# Collaboration endpoint active
curl -s http://localhost:8888/api/collaboration/room/
# Expected: "Can \"Upgrade\" only to \"WebSocket\"."

# Server extensions
ov shell jupyter-colab -c "jupyter server extension list 2>&1 | grep ydoc"
# Expected: jupyter_server_ydoc enabled OK

# Versions
ov shell jupyter-colab -c "python -c 'import jupyterlab; print(jupyterlab.__version__)'"
ov shell jupyter-colab -c "python -c 'import jupyter_collaboration; print(jupyter_collaboration.__version__)'"
```

## Testing Collaboration

To test real-time collaboration, deploy `sway-browser-vnc` alongside:

```bash
ov start sway-browser-vnc
# Open JupyterLab in two Chrome tabs via container DNS:
ov cdp open sway-browser-vnc "http://ov-jupyter-colab:8888/lab"
# Open second tab
ov cdp open sway-browser-vnc "http://ov-jupyter-colab:8888/lab"
```

**Executing cells via CDP:** Use `Input.dispatchKeyEvent` (not VNC keys — unreliable when Chrome lacks compositor focus):

```bash
TAB=<tab-id>
# Focus cell, then Shift+Enter via CDP
ov cdp eval sway-browser-vnc $TAB "document.querySelector('.jp-Cell-inputArea .cm-content')?.focus()"
ov cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"rawKeyDown","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
ov cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"keyUp","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
```

## Differences from jupyter Image

| | jupyter-colab | jupyter |
|---|---|---|
| Base | fedora | nvidia |
| GPU | None | CUDA |
| Platforms | amd64 + arm64 | amd64 only |
| Focus | Collaboration | ML/AI |
| Size | ~3.4 GB | ~15+ GB |

## When to Use This Skill

**MUST be invoked** when the task involves the jupyter-colab image, collaborative Jupyter notebooks, or lightweight Jupyter deployments without GPU. Invoke this skill BEFORE reading source code or launching Explore agents.
