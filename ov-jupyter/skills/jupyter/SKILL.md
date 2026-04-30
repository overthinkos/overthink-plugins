---
name: jupyter
description: |
  Lightweight JupyterLab with real-time collaboration on port 8888. No GPU required.
  Based on fedora (not nvidia), supports both amd64 and arm64.
  MUST be invoked before building, deploying, configuring, or troubleshooting the jupyter image.
---

# jupyter

Lightweight JupyterLab with real-time collaboration via jupyter-collaboration (Y-CRDT).

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, jupyter (sub-layers: jupyter-mcp), notebook-templates, dbus, ov |
| Platforms | linux/amd64, linux/arm64 |
| Ports | 8888 |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

1. `fedora` (Fedora 43 base — no GPU)
2. `pixi` → `python` → `supervisord` (transitive)
3. `jupyter` — JupyterLab + jupyter-collaboration + data science (composes `jupyter-mcp` sub-layer for MCP extension)
4. `notebook-templates` — Starter notebooks (data layer, seeds ~/workspace)
5. `agent-forwarding` — SSH/GPG agent forwarding
5. `dbus` — D-Bus session bus
6. `ov` — ov CLI binary

## Ports

| Port | Service | Protocol |
|------|---------|----------|
| 8888 | JupyterLab | HTTP |

## Volumes

| Name | Path | Purpose |
|------|------|---------|
| workspace | ~/workspace | Persistent notebook storage |

## Quick Start

```bash
ov image build jupyter
ov config jupyter
ov start jupyter
# Open http://localhost:8888
```

Routine ops once started:

```bash
ov status jupyter         # state + tools probe (dbus / ov / supervisord)
ov restart jupyter        # atomic systemctl --user restart (preserves
                          # ExecStopPost → ExecStartPost order — important
                          # when the deploy carries a tailscale tunnel)
ov stop jupyter           # stop without restarting
ov logs jupyter           # supervisord aggregated stdout/stderr
```

## What's Installed

Beyond the JupyterLab core, the image's pixi env carries: pandas /
polars / numpy / scipy / scikit-learn / matplotlib / seaborn for data
work; pyarrow / duckdb for column-store interop; **spacy 3.8.x +
`en_core_web_sm`** for NLP (tokenization, NER, POS, dependency
parsing); black / pytest / graphviz / pyyaml / tqdm as utilities. Full
list and version pins live in `layers/jupyter/pixi.toml` — see
`/ov-jupyter:jupyter` for the package matrix.

## Key Layers

- `/ov-jupyter:jupyter` — JupyterLab + collaboration + data science
- `/ov-foundation:agent-forwarding` — SSH/GPG forwarding

## Related Images

- `/ov-jupyter:jupyter-ml` — GPU-accelerated variant (nvidia base, full ML stack)
- `/ov-jupyter:jupyter-ml-notebook` — GPU variant with fine-tuning notebooks
- `/ov-openwebui:openwebui` — Open WebUI consumes jupyter for code execution and MCP tools
- `/ov-hermes:hermes` — Hermes agent consumes jupyter MCP for notebook manipulation
- `/ov-foundation:fedora` — parent base image

## MCP Server (Programmatic Notebook Access)

The image includes a built-in MCP server at `http://localhost:8888/mcp` (Streamable HTTP transport, MCP spec 2025-11-25). AI agents can create, read, edit, execute, and watch notebooks programmatically — with changes syncing live to all collaborators via CRDT.

The MCP server name is set via the `MCP_SERVER_NAME` environment variable (default: `jupyter`). For multi-instance deployments, override per-instance: `ov config jupyter -i work -e MCP_SERVER_NAME=jupyter-work`.

### Configure Claude Code

There are three supported ways to give Claude Code access to the
jupyter MCP server. Pick whichever matches the scope you want:

**Project-scoped `.mcp.json` (preferred for `claude -p` from a
specific working directory).** Drop a file at `<project-root>/.mcp.json`
with the canonical shape:

```json
{
  "mcpServers": {
    "jupyter": {
      "type": "http",
      "url": "http://localhost:8888/mcp"
    }
  }
}
```

Whenever `claude` (interactive) or `claude -p` (print mode) launches
from that project root, it discovers the file and registers `jupyter`
as an HTTP MCP server. Typical layout: a workspace dir on the host
that is also bind-mounted as the jupyter `workspace` volume — running
`claude -p` from inside that dir lets the model both see the notebook
files on disk AND drive jupyter via MCP. (See `/ov-core:deploy` for the
`--bind workspace=…` flag that pins the volume to a host path.)

**`claude mcp add` shorthand** — equivalent to writing the file
yourself:

```bash
claude mcp add --transport http --scope project jupyter http://localhost:8888/mcp
```

**The `ov-jupyter` plugin** (`plugins/ov-jupyter/.mcp.json`) — declares
the same server at the project level for the overthink repo itself.
Suitable when you're working IN the overthink checkout (the file is
gitignored downstream of `.claude-plugin/plugin.json`).

**Container must be running BEFORE `claude` starts.** Claude Code
discovers MCP servers at session-start time. If `ov-jupyter.service`
is not running when `claude` launches, the server registration shows
"failed to connect" and `claude` will not auto-reconnect mid-session.
Start jupyter first, then start the claude session:

```bash
ov status jupyter | grep -q running || ov start jupyter
sleep 3                             # let supervisord settle
cd <workspace-with-.mcp.json>
claude                              # or claude -p "..."
```

If you start `claude` first and then `ov start jupyter`, you'll need
to exit and restart the claude session before the jupyter MCP tools
become available.

### 13 MCP tools

| Category | Tools |
|----------|-------|
| Notebook management | `list_notebooks`, `get_notebook`, `create_notebook` |
| Cell operations (CRDT) | `get_cell`, `update_cell`, `insert_cell`, `delete_cell`, `execute_cell` |
| Room management | `open_notebook_session`, `close_notebook_session` |
| Change watching | `watch_notebook` (blocks until change or timeout) |
| Collaboration | `get_active_users`, `get_active_sessions` |

See `/ov-jupyter:jupyter` for full parameter and return type documentation.

### Testing with `claude -p`

```bash
# Prerequisites: ov start jupyter

# Create and work with a notebook
claude -p "Call create_notebook with path 'test.ipynb'"
claude -p "Call open_notebook_session with path 'test.ipynb'"
claude -p "Call insert_cell with path 'test.ipynb', index 0, source 'print(42)', cell_type 'code'"
claude -p "Call execute_cell with path 'test.ipynb' and index 0"
claude -p "Call close_notebook_session with path 'test.ipynb'"
```

### Multi-client collaboration

Multiple `claude -p` sessions (or any MCP client) can edit the same notebook simultaneously. Each session creates a separate HTTP connection but shares the same CRDT document. Changes from one client are immediately visible to all others.

```bash
# Client A watches for changes
claude -p "Call watch_notebook with path 'test.ipynb' and timeout 30" &

# Client B makes a change
claude -p "Call update_cell with path 'test.ipynb', index 0, source 'print(\"hello\")'"

# Client A's watch_notebook returns: {"changed": true, "cell_count": 1}
```

## Verification

After `ov start` (or `ov restart` to recycle a running container):

```bash
# Container and services running
ov status jupyter
ov service status jupyter

# Atomic restart (preserves ExecStopPost → ExecStartPost order; important
# when the deploy carries a tailscale tunnel — the off/on sequence
# must be tight to avoid a serve-without-listener window)
ov restart jupyter

# JupyterLab responds
curl -s -o /dev/null -w '%{http_code}' http://localhost:8888    # 200

# MCP server responds
curl -s http://localhost:8888/mcp -X POST \
  -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"test","version":"0.1.0"}}}'
# Expected: SSE response with serverInfo.name = "jupyter"

# Server extensions
ov shell jupyter -c "jupyter server extension list 2>&1 | grep -E 'ydoc|colab_mcp'"
# Expected: jupyter_server_ydoc enabled OK, jupyter_mcp 0.1.0 OK

# In-container PATH includes the pixi env (post-2026-04-29 fix —
# pixi runtime env now flows through the BUILDER, not the LAYER)
ov shell jupyter -c 'echo PATH=$PATH'
# Expected: PATH starts with /home/user/.pixi/bin:/home/user/.pixi/envs/default/bin:...

# spacy + en_core_web_sm load (the build-scope eval check, runnable
# at runtime too)
ov shell jupyter -c '~/.pixi/envs/default/bin/python -c "import spacy; spacy.load(\"en_core_web_sm\"); print(\"ok\")"'
# Expected: ok

# MCP tools via Claude Code (run from a dir with .mcp.json)
claude mcp list    # Should show: jupyter: http://localhost:8888/mcp (HTTP) - ✓ Connected
```

## Testing Collaboration

To test real-time collaboration, deploy `sway-browser-vnc` alongside:

```bash
ov start sway-browser-vnc
# Open JupyterLab in two Chrome tabs via container DNS:
ov eval cdp open sway-browser-vnc "http://ov-jupyter:8888/lab"
# Open second tab
ov eval cdp open sway-browser-vnc "http://ov-jupyter:8888/lab"
```

**Executing cells via CDP:** Use `Input.dispatchKeyEvent` (not VNC keys — unreliable when Chrome lacks compositor focus):

```bash
TAB=<tab-id>
# Focus cell, then Shift+Enter via CDP
ov eval cdp eval sway-browser-vnc $TAB "document.querySelector('.jp-Cell-inputArea .cm-content')?.focus()"
ov eval cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"rawKeyDown","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
ov eval cdp raw sway-browser-vnc $TAB 'Input.dispatchKeyEvent' '{"type":"keyUp","windowsVirtualKeyCode":13,"nativeVirtualKeyCode":13,"modifiers":8}'
```

## Test Coverage

Latest `ov test jupyter` run: **29 passed, 0 failed, 0 skipped**.
All tests embedded in the `org.overthinkos.eval` OCI label:
jupyter-lab binary under pixi, notebook-templates provisioned into
`${HOME}/workspace`, jupyter-mcp extension enabled, fastmcp pip
package installed. Deploy-scope: supervisord up, port 8888 reachable
on `127.0.0.1`, `/api` returns 200 with `version` in body, `/mcp`
returns 400 on empty POST (proving MCP routing is wired). Image-scope:
`jupyter_mcp` appears in extension list, workspace has ≥1 `.ipynb`.

See `/ov-build:eval` for the framework and author-facing gotchas.

## Related Skills

- `/ov-jupyter:jupyter`, `/ov-jupyter:jupyter-mcp`, `/ov-jupyter:notebook-templates`
- `/ov-build:eval` — declarative testing framework
- `/ov-core:config` — deploy setup
- `/ov-build:mcp` — the image inherits 3 deploy-scope `mcp:` declarative checks from the `jupyter` layer (`ping`, `list-tools` asserting `insert_cell`/`execute_cell`, `call list_notebooks`). Run `ov test jupyter --filter mcp` to exercise them against a live deployment, or `ov eval mcp list-tools jupyter` for ad-hoc inspection
- `/ov-jupyter:jupyter-ml`, `/ov-jupyter:jupyter-ml-notebook` — GPU variants that inherit the same MCP test suite

## When to Use This Skill

**MUST be invoked** when the task involves the jupyter image, collaborative Jupyter notebooks, lightweight Jupyter deployments without GPU, MCP-based notebook access, or multi-client collaboration. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov-build:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov-build:build` — `build.yml` vocabulary (distros, builders, init-systems)
