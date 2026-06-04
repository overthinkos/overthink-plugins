---
name: hermes-full-layer
description: |
  Hermes agent with AI CLIs (Claude Code, Codex, Gemini), developer tools, DevOps tools, and ov.
  Use when working with the hermes-full metalayer or full-featured standalone hermes deployments.
---

# Layer: hermes-full

Metalayer composing Hermes AI agent with a complete tool suite for standalone deployment. No browser included — use with `selkies-desktop` via cross-container `env_provide: BROWSER_CDP_URL` for shared browser automation.

## Composition

```yaml
layers:
  - hermes          # AI agent with browser tools, MCP, LLM auto-config
  - claude-code     # Anthropic Claude Code CLI
  - codex           # OpenAI Codex CLI
  - gemini          # Google Gemini CLI
  - dev-tools       # bat, ripgrep, neovim, gh, direnv, fd-find, htop, etc.
  - devops-tools    # AWS CLI, Scaleway, kubectx, OpenTofu, wrangler, jq, rsync
  - ov              # Overthink CLI for in-container management
  - tmux            # Terminal multiplexer for persistent sessions
```

## Browser Integration

The hermes layer declares `env_accept: BROWSER_CDP_URL`. When deployed alongside a `selkies-desktop` container, the chrome layer's `env_provide` injects `BROWSER_CDP_URL=http://ov-selkies-desktop:9222` into the hermes quadlet via `ov config --update-all`. Hermes browser tools (`browser_navigate`, `browser_click`, `browser_snapshot`) then control the desktop Chrome across the container network.

Without a browser provider, hermes browser tools fall back to local headless mode (requires `hermes-playwright` layer) or are unavailable.

## Usage

```yaml
# image.yml
hermes:
  base: fedora
  layers:
    - agent-forwarding
    - hermes-full
    - dbus
```

## Related Layers

- `/ov-hermes:hermes` — Core Hermes agent (LLM providers, MCP, browser dispatch)
- `/ov-coder:claude-code` — Anthropic Claude Code CLI
- `/ov-coder:codex` — OpenAI Codex CLI
- `/ov-coder:gemini` — Google Gemini CLI
- `/ov-coder:dev-tools` — Developer CLI utilities
- `/ov-coder:devops-tools` — Cloud and infrastructure tools
- `/ov-tools:ov` — Overthink CLI binary
- `/ov-infrastructure:tmux-layer` — Terminal multiplexer for persistent sessions (`ov tmux` commands)
- `/ov-selkies:chrome` — Provides `BROWSER_CDP_URL` (cross-container, from selkies-desktop)
- `/ov-selkies:chrome-devtools-mcp` — Chrome DevTools MCP server (auto-discovered via `mcp_provide`, 29 tools)
- `/ov-jupyter:jupyter-mcp` — JupyterLab CRDT MCP server (auto-discovered via `mcp_provide`, 11 tools: notebook_*/cell_* + notebook_list_users + room_list; auto-attach single-room invariant)
- `/ov-build:ov-mcp-cmd` — host-side MCP client (`ov eval mcp ping|list-tools|call|...`) to verify either of the above is alive and exposing the expected tool catalog before hermes tries to invoke them

## Related Images

- `/ov-hermes:hermes` — Standalone full-featured hermes image
- `/ov-hermes:hermes-playwright` — Hermes with local Playwright Chromium

## When to Use This Skill

Use when working with the `hermes-full` metalayer, full-featured standalone hermes deployments, or the composition of hermes with AI CLIs and developer tools.

## Related

- `/ov-image:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-eval:eval` — declarative testing (`eval:` block, `ov eval image`, `ov eval live`)
