---
name: openclaw-full-layer
description: |
  Maximal OpenClaw deployment (gateway + browser + all feasible tools/skills).
  Use when working with the openclaw-full layer.
---

# openclaw-full -- Maximal OpenClaw deployment

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `candy.yml` (metalayer, layers only) |
| Depends | none (composes other layers) |

## Composed Layers

This metalayer includes the following 25 layers:

- `openclaw` -- AI gateway
- `chrome` -- Google Chrome browser
- `claude-code` -- Claude Code CLI
- `codex` -- OpenAI Codex CLI
- `gemini` -- Google Gemini CLI
- `clawhub` -- ClawHub skill registry CLI
- `mcporter` -- MCP server CLI
- `oracle` -- Prompt bundling CLI
- `xurl` -- X/Twitter API CLI
- `summarize` -- URL/transcript extractor
- `playwright` -- Browser automation
- `blogwatcher` -- Blog/RSS feed monitor
- `gifgrep` -- GIF search and download
- `wacli` -- WhatsApp CLI
- `goplaces` -- Google Places API CLI
- `songsee` -- Audio spectrogram
- `sag` -- ElevenLabs TTS CLI
- `camsnap` -- Camera snapshot CLI
- `gogcli` -- Google Workspace CLI
- `ordercli` -- Food delivery order CLI
- `himalaya` -- Email CLI
- `uv` -- Python package manager
- `nano-pdf` -- PDF editing CLI
- `gh` -- GitHub CLI
- `tmux` -- Terminal multiplexer
- `ffmpeg` -- FFmpeg multimedia
- `ripgrep` -- Fast text search
- `sqlite` -- SQLite database CLI

## Usage

```yaml
# box.yml
layers:
  - openclaw-full
```

## Used In Images

- `openclaw-full`

## Related Layers
- `/charly-openclaw:openclaw` — base gateway layer included in this metalayer
- `/charly-selkies:sway-desktop-vnc` — desktop layer paired in sway-browser images

## Related Commands
- `/charly-automation:openclaw-deploy` — gateway configuration, model auth, and channel setup
- `/charly-build:build` — builds the metalayer composition into an image
- `/charly-core:charly-config` — sets up secrets and quadlets for openclaw-full images

## When to Use This Skill

Use when the user asks about:
- Full OpenClaw deployment with all tools
- The `openclaw-full` metalayer composition
- Which tools are included in the maximal OpenClaw image
- The `openclaw-full` layer

## Related

- `/charly-image:layer` — layer authoring reference (`candy.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
