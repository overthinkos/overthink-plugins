---
name: hermes-full
description: |
  Hermes agent with AI CLIs (Claude Code, Codex, Gemini), developer tools, DevOps tools, and ov.
  Use when working with the hermes-full metalayer or full-featured standalone hermes deployments.
---

# Layer: hermes-full

Metalayer composing Hermes AI agent with a complete tool suite for standalone deployment. No browser included — use with `selkies-desktop` via cross-container `env_provides: BROWSER_CDP_URL` for shared browser automation.

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

The hermes layer declares `env_accepts: BROWSER_CDP_URL`. When deployed alongside a `selkies-desktop` container, the chrome layer's `env_provides` injects `BROWSER_CDP_URL=http://ov-selkies-desktop:9222` into the hermes quadlet via `ov config --update-all`. Hermes browser tools (`browser_navigate`, `browser_click`, `browser_snapshot`) then control the desktop Chrome across the container network.

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

- `/ov-layers:hermes` — Core Hermes agent (LLM providers, MCP, browser dispatch)
- `/ov-layers:claude-code` — Anthropic Claude Code CLI
- `/ov-layers:codex` — OpenAI Codex CLI
- `/ov-layers:gemini` — Google Gemini CLI
- `/ov-layers:dev-tools` — Developer CLI utilities
- `/ov-layers:devops-tools` — Cloud and infrastructure tools
- `/ov-layers:ov` — Overthink CLI binary
- `/ov-layers:tmux` — Terminal multiplexer for persistent sessions (`ov tmux` commands)
- `/ov-layers:chrome` — Provides `BROWSER_CDP_URL` (cross-container, from selkies-desktop)
- `/ov-layers:chrome-devtools-mcp` — Chrome DevTools MCP server (auto-discovered via `mcp_provides`, 29 tools)
- `/ov-layers:jupyter-mcp` — JupyterLab CRDT MCP server (auto-discovered via `mcp_provides`, 13 tools)
- `/ov:mcp` — host-side MCP client (`ov eval mcp ping|list-tools|call|...`) to verify either of the above is alive and exposing the expected tool catalog before hermes tries to invoke them

## Related Images

- `/ov-images:hermes` — Standalone full-featured hermes image
- `/ov-images:hermes-playwright` — Hermes with local Playwright Chromium

## When to Use This Skill

Use when working with the `hermes-full` metalayer, full-featured standalone hermes deployments, or the composition of hermes with AI CLIs and developer tools.

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
