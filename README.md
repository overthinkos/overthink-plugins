# Overthink Plugins

Claude Code plugins for Overthink — the container management experience for you and your AI.

## Architecture (v2.0.0 — 2026-04-30)

The marketplace is organized into **three concentric tiers**:

1. **Core verbs** — daily-ops CLI knowledge (`ov-core`, `ov-build`, `ov-advanced`).
   Pick the tiers that match what you do.
2. **Per-pod plugins** — self-contained image+layer skills for each shipped
   pod family. Pick the pods you actually run.
3. **Foundation reference** — comprehensive maintainer catalog of base
   distros and shared layers (`ov-foundation`). For layer/image authors.

Plus three special-purpose plugins: `ov-dev` (contributor tooling),
`ov-vms` (libvirt-managed VMs).

## Which plugins to enable

| Use case | Enable |
|---|---|
| Single-pod jupyter project | `ov-core` + `ov-jupyter` |
| Single-pod coder/dev pod | `ov-core` + `ov-coder` |
| Multi-pod home AI server | `ov-core` + per-pod plugins for each pod you run |
| Layer/image authoring | `ov-core` + `ov-build` + `ov-foundation` |
| Power user (k8s / VMs / probes) | `ov-core` + `ov-advanced` |
| Maintainer / full coverage | enable everything |

Project `.claude/settings.json` example for a single-jupyter consumer:

```json
{
  "enabledPlugins": {
    "ov-core@ov-plugins": true,
    "ov-jupyter@ov-plugins": true
  }
}
```

## Plugins

### Core verbs

| Plugin | Skills | Audience | Description |
|---|---:|---|---|
| `ov-core` | 14 | Every consumer | Daily ops: `start`, `stop`, `status`, `logs`, `shell`, `cmd`, `config`, `deploy`, `update`, `remove`, `service`, `ssh`, `version`, `doctor` |
| `ov-build` | 15 | Layer/image authors | Authoring: `build`, `image`, `layer`, `generate`, `validate`, `inspect`, `list`, `merge`, `new`, `mcp`, `pull`, `secrets`, `settings`, `eval`, `migrate` |
| `ov-advanced` | 18 | Power users | k8s + VM + probe verbs: `kubernetes`, `eval-k8s`, `vm`, `libvirt`, `host-deploy`, `enc`, `openclaw`, `alias`, `udev`, `sidecar`, `tmux`, `cdp`, `wl`, `dbus`, `vnc`, `spice`, `record`, `wl-overlay` |

### Per-pod plugins

Each per-pod plugin is **self-contained**: it ships the image SKILL(s),
every layer SKILL the image composes, optional data/template layer
SKILLs, and (for MCP-providing pods) a `.mcp.json` registering the
pod's MCP server. Enable a per-pod plugin to get all the knowledge
needed to run that pod.

| Plugin | Image families | MCP server (port) | Description |
|---|---|---|---|
| `ov-jupyter` | jupyter, jupyter-ml, jupyter-ml-notebook, unsloth-studio | jupyter @ 8888 | JupyterLab + ML stack + 13 CRDT MCP tools for notebook manipulation |
| `ov-coder` | arch-coder, debian-coder, fedora-coder, ubuntu-coder, arch-ov | ov @ 18765 | Developer pods (Claude Code, Codex, Gemini, ForgeCode, Oracle, kubectl, docker, gcloud) with the ov MCP server |
| `ov-selkies` | selkies-desktop\*, sway-browser-vnc | chrome-devtools @ 9224 | Wayland desktop in a container (sway/niri/labwc + VNC + Chrome) |
| `ov-openclaw` | openclaw\* | chrome-devtools @ 9224 | Multi-variant AI workstation (bootc, full, ml, sway, ollama, browser) |
| `ov-ollama` | ollama | — | Ollama LLM server. Pair with `ov-jupyter` for in-notebook inference |
| `ov-openwebui` | openwebui | — (consumes jupyter MCP) | Web chat frontend with code-execution via jupyter |
| `ov-comfyui` | comfyui | — | Stable-Diffusion / image-generation node graph |
| `ov-immich` | immich, immich-ml | — | Self-hosted photo-management |
| `ov-hermes` | hermes, hermes-playwright | — (consumes jupyter MCP) | Agent runtime |
| `ov-filebrowser` | filebrowser | — | Web-based file management |

### Foundation reference

| Plugin | Skills | Description |
|---|---:|---|
| `ov-foundation` | ~80 | Comprehensive maintainer catalog: base distros (fedora, archlinux, debian, ubuntu, aurora, bazzite-ai), foundation layers (pixi, supervisord, cuda, nvidia, agent-forwarding, dbus, ov, secrets, k3s, tailscale, traefik, postgresql, valkey, …), all the layers and images that aren't owned by a specific pod plugin |

### Special-purpose

| Plugin | Skills | Description |
|---|---:|---|
| `ov-dev` | 3 + 3 agents | Repo contributor tooling: skill maintenance, host-infra, install-plan, root-cause-analyzer agent, layer-validator agent, testing-validator agent |
| `ov-vms` | 6 | libvirt-managed VMs: vm-deploy-target, cutover-policy, etc. |

## How a per-pod plugin is built

Take `ov-jupyter` as the canonical example:

```
ov-jupyter/
  README.md
  .claude-plugin/
    plugin.json              # name, description, mcpServers ref
  .mcp.json                  # jupyter MCP server registration
  skills/
    jupyter/SKILL.md           # image-level skill — "what is this image"
    jupyter-ml/SKILL.md        # variant
    jupyter-ml-notebook/SKILL.md
    unsloth-studio/SKILL.md
    jupyter-layer/SKILL.md     # layer-level — "how is jupyter composed"
    jupyter-ml-layer/SKILL.md  # (renamed to avoid collision with image SKILL)
    jupyter-mcp/SKILL.md       # the MCP extension layer
    notebook-templates/SKILL.md
    notebook-finetuning/SKILL.md
    llama-cpp/SKILL.md         # composed by jupyter-ml
    unsloth/SKILL.md
    …
```

When a consumer enables `ov-jupyter@ov-plugins: true`, every SKILL.md
in `skills/` becomes available to Claude's dispatcher, AND the
`.mcp.json` registers the running jupyter MCP server.

## Adding a new pod plugin

When a new pod ships:

1. `mkdir -p plugins/ov-<pod>/{.claude-plugin,skills}`
2. Author the image and layer SKILLs directly inside that directory.
3. If the image provides an MCP server, add `.mcp.json` and reference
   it from `plugin.json` via `"mcpServers": "./.mcp.json"`.
4. Add the entry to `plugins/.claude-plugin/marketplace.json`.
5. Update this README's table.

## Naming convention

- Plugin name: `ov-<pod>` for per-pod plugins, `ov-<tier>` for verb tiers (`ov-core`, `ov-build`, `ov-advanced`), `ov-foundation` for the catalog.
- Skill names within a plugin: prefer the layer/image's own name. When an image and a layer share a name (e.g. `jupyter`), rename the layer copy to `<name>-layer`.
- Cross-references: `/ov-<plugin>:<skill>` from any SKILL.md to any other.
