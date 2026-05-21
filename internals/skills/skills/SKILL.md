---
name: skills
description: |
  Skill maintenance guidelines: when and how to update skills, CLAUDE.md, and README.md.
  Use when updating documentation, feeding back operational insights, or auditing skill coverage.
---

# Skill Maintenance Guidelines

## Overview

Skills are living documents at `plugins/<plugin>/skills/<name>/SKILL.md`. They are the primary knowledge source for Claude Code — always invoked before codebase exploration. This skill covers when and how to update them.

## Skill Dispatcher (authoritative in CLAUDE.md R0; mirrored here)

CLAUDE.md R0 (SKILLS FIRST — THE SUPREME RULE) is the authoritative dispatcher. This section mirrors it so skill-authors already inside `/ov-internals:skills` see the same mapping without having to context-switch back to CLAUDE.md. If this table ever drifts from CLAUDE.md R0, **CLAUDE.md R0 wins** — fix this file, never the other way around.

| Trigger | Skills to load BEFORE doing anything |
|---|---|
| `ov update` / `ov vm *` / VM entities | `/ov-vm:vm` + `/ov-internals:vm-deploy-target` |
| `ov deploy add/del` / pod or container deploys | `/ov-core:deploy` |
| host-target / nested host deploy | `/ov-local:local-deploy` + `/ov-internals:local-infra` |
| `ov eval live` / `ov eval cdp/wl/dbus/vnc/mcp/record/spice/libvirt` | `/ov-eval:eval` |
| `ov eval k8s <verb>` | `/ov-kubernetes:eval-k8s` |
| Editing `layer.yml` / layer authoring | `/ov-image:layer` |
| Editing `image.yml` / image composition | `/ov-image:image` |
| `ov image build` / `ov image generate` / Containerfile | `/ov-build:build` + `/ov-build:generate` + `/ov-internals:generate-source` |
| `ov image validate` / schema error | `/ov-build:validate` |
| Secret management / `ov secrets` / Secret Service / GPG `.secrets` | `/ov-build:secrets` |
| Schema v4 migration / legacy → new format | `/ov-build:migrate` |
| Hard-cutover concerns / rename sweeps | `/ov-internals:cutover-policy` |
| `disposable: true` semantics | `/ov-internals:disposable` |
| Go source work | `/ov-internals:go` |
| IR / InstallPlan / DeployTarget / OCITarget | `/ov-internals:install-plan` |
| OCI labels / capabilities contract | `/ov-internals:capabilities` |
| VmSpec / libvirt / cloud-init / OVMF | `/ov-internals:vm-spec` + renderer skills |
| Unexpected failure / anomaly | `/ov-internals:root-cause-analyzer` agent |
| Engineering-discipline trigger (failure / dup pattern / ad-hoc fix / "out of scope" framing) | `/ov-internals:strict-policy` |
| "What does layer X do?" — pod-specific | `/ov-jupyter:<name>`, `/ov-coder:<name>`, `/ov-selkies:<name>`, `/ov-openclaw:<name>`, `/ov-ollama:<name>`, `/ov-openwebui:<name>`, `/ov-comfyui:<name>`, `/ov-immich:<name>`, `/ov-hermes:<name>`, `/ov-filebrowser:<name>` |
| "What does layer X do?" — base distros / GPU / bootc | `/ov-distros:<name>` (archlinux, fedora, debian, ubuntu, nvidia, cuda, rocm, bootc-base, …) |
| "What does layer X do?" — language runtime | `/ov-languages:<name>` (python, python-ml, pixi) |
| "What does layer X do?" — infrastructure service | `/ov-infrastructure:<name>` (postgresql, redis, k3s, traefik, supervisord, tailscale, gocryptfs, virtualization, dbus-layer, tmux-layer, …) |
| "What does layer X do?" — CLI utility | `/ov-tools:<name>` (ripgrep, himalaya, whisper, ov, ov-full, …) |
| Skill authoring / maintenance | `/ov-internals:skills` (this skill) |

If multiple triggers apply, load ALL matching skills in ONE message (parallel `Skill` calls). Full index: `plugins/README.md` (250+ skills).

## When to Update Skills

| Trigger | Action |
|---------|--------|
| Deployment step fails or needs undocumented workaround | Update the relevant `/ov-core:*`, `/ov-build:*`, `/ov-eval:*`, `/ov-automation:*`, kind plugin (`/ov-image:*`, `/ov-vm:*`, `/ov-kubernetes:*`, `/ov-local:*`, `/ov-pod:*`), per-pod plugin (`/ov-jupyter:*`, `/ov-coder:*`, …), or split-foundation plugin (`/ov-distros:*`, `/ov-languages:*`, `/ov-infrastructure:*`, `/ov-tools:*`) |
| Verification check missing from image skill | Add to the image skill's Verification section |
| Skill's recommended defaults are wrong | Fix in the skill, not CLAUDE.md |
| New feature added to ov CLI | Update `/ov-core:<cmd>` or `/ov-build:<cmd>` skill + `/ov-internals:go` source map |
| New layer or image added | Create skill via `ov image new layer` scaffold or manual SKILL.md |
| Bug fix changes behavior | Document the fix in affected skills |
| Cross-skill behavior discovered | Update Cross-References in all affected skills |
| Removed identifier still referenced in skill paragraph (R5 self-test failed) | Update / delete the paragraph in the SAME commit as the removal (R5) |

## When NOT to Update Skills

- **Ephemeral issues** — use conversation context or memory
- **User-specific config** — use Claude Code memory system
- **Bug fixes in ov code** — the fix is in git; document behavioral changes in skills only
- **Anything derivable from code** — skills document *usage*, not implementation details

## How to Update

1. Edit the skill file at `plugins/<plugin>/skills/<skill-name>/SKILL.md`
2. If the insight affects cross-skill behavior, update CLAUDE.md too
3. After any non-trivial deployment session, ask: "Did we learn anything that future sessions should know?"

## Skill File Structure

```markdown
---
name: <skill-name>
description: |
  One-sentence description of when to invoke this skill.
  For layers: "Use when working with <component>."
  For images: "MUST be invoked before building, deploying, or troubleshooting the <image> image."
---

# <Title>

## Overview / Properties
## Key Sections (varies by type)
## Cross-References (related skills)
## When to Use This Skill
```

## CLAUDE.md vs Skills

| Content type | Where it belongs |
|-------------|-----------------|
| Project philosophy, architecture, key rules | CLAUDE.md |
| Command usage, flags, examples | `/ov-core:<cmd>` or `/ov-build:<cmd>` skill |
| Layer properties, packages, ports | per-pod plugin (`/ov-jupyter:<name>`, `/ov-coder:<name>`, …) or split-foundation plugin (`/ov-distros:*`, `/ov-languages:*`, `/ov-infrastructure:*`, `/ov-tools:*`) for base layers |
| Image composition, deployment, verification | per-pod plugin or `/ov-distros:<name>` / `/ov-infrastructure:<name>` for base images |
| Skill disambiguation (which skill to use) | CLAUDE.md (brief table) |
| Detailed operational patterns | Relevant `/ov-core:*` / `/ov-build:*` / `/ov-eval:*` / `/ov-automation:*` / kind-plugin skill |

## Command skills vs topic skills

Most skills under `plugins/ov-core/skills/` and `plugins/ov-build/skills/`
map 1:1 to a top-level ov command (e.g. `/ov-build:build` ↔ `ov image build`,
`/ov-core:ov-status` ↔ `ov status`).
**Topic skills** are the exception: they don't correspond to a
top-level command but cover a cross-cutting concept surfaced by flags
or layer composition. Today's topic skills:

| Skill | Surfaced via | What it covers |
|---|---|---|
| `/ov-automation:enc` | `ov config --encrypt`, `ov config mount`, `ov config unmount`, `ov config passwd` | Encrypted-volume (gocryptfs) semantics, keyring resolution, `ov-enc-<image>-<volume>.scope` lifecycle |
| `/ov-automation:openclaw-deploy` | Composing `openclaw-*` layers | OpenClaw AI gateway deployment story |
| `/ov-automation:sidecar` | `ov config --sidecar tailscale` | Sidecar-container model, pod networking, env-var routing |

When adding a new command, always create a matching command skill. Consider a topic skill when a concept spans multiple commands or layers and the natural home isn't any single command's skill. Keep the frontmatter `description:` explicit about the topic nature (the blocking `Skill:` tool dispatcher matches on description keywords).

## Plugin Structure (post 2026-05-XX use-case reorganization)

Plugins are sorted into four use-case buckets. Directory names live at
`plugins/<name>/` (no `ov-` prefix); plugin.json `name:` fields keep the
`ov-` prefix; every skill is invoked as `/ov-<plugin>:<skill>`.

| Bucket | Plugin | Skills | Purpose |
|---|---|---:|---|
| commands | `ov-core` | 14 | Lifecycle verbs (start/stop/status/logs/shell/ssh/deploy/update/...) |
| commands | `ov-build` | 12 | Build/authoring verbs (build/generate/list/inspect/merge/new/pull/validate/secrets/settings/migrate/mcp) |
| commands | `ov-eval` | 9 | `ov eval` orchestrator + live probes (cdp/wl/wl-overlay/dbus/vnc/spice/libvirt/record) |
| commands | `ov-automation` | 6 | tmux verb, host-side helpers (alias/udev), topic flags (enc/sidecar/openclaw-deploy) |
| kind | `ov-image` | 2 | `kind: image` and `kind: layer` schema reference |
| kind | `ov-vm` | 7 | `kind: vm` schema + bootc VM catalog |
| kind | `ov-kubernetes` | 2 | `kind: k8s` schema + cluster probes |
| kind | `ov-local` | 2 | `kind: local` schema + ssh-host deploys |
| kind | `ov-pod` | 1 | `kind: pod` and `kind: deploy` schema (thin pointer) |
| development | `ov-internals` | 14 + 3 agents | Go source / IR / capabilities / vm-spec / renderers / cutover-policy / strict-policy / disposable + enforcement agents + github MCP |
| images | `ov-distros` | 34 | Base OS, GPU runtime, bootc, distro builders |
| images | `ov-languages` | 4 | python, python-ml, pixi |
| images | `ov-infrastructure` | 22 | postgres, redis, k3s, traefik, supervisord, tailscale, gocryptfs, virtualization, dbus-layer, tmux-layer, ... |
| images | `ov-tools` | 19 | CLI utilities + ov binary deploy |
| images | `ov-jupyter` | 15 | jupyter image family + jupyter MCP @ 8888 |
| images | `ov-coder` | 33 | coder/dev images + ov MCP @ 18765 |
| images | `ov-selkies` | 41 | selkies-desktop family + chrome-devtools MCP @ 9224 |
| images | `ov-openclaw` | 12 | openclaw AI workstation + chrome-devtools MCP @ 9224 |
| images | `ov-versa` | 9 | versa image — marimo + airflow + OSM analytics + 2 MCP servers |
| images | `ov-ollama` | 2 | ollama LLM-server image |
| images | `ov-openwebui` | 2 | openwebui chat frontend |
| images | `ov-comfyui` | 2 | comfyui image generation |
| images | `ov-immich` | 4 | immich photo management |
| images | `ov-hermes` | 6 | hermes agent image |
| images | `ov-filebrowser` | 2 | filebrowser web file management |

## Two-Layer Sync Architecture

The project uses two complementary sync mechanisms with `.gitignore` as the
boundary:

| What | Synced by | Visibility |
|------|-----------|------------|
| Code, CLAUDE.md, skills, layers, images | Git | Public (committed) |
| `.claude/memory/` (auto-memory) | Syncthing | Private (gitignored) |
| `.claude/settings.local.json` (personal overrides) | Syncthing | Private (gitignored) |
| `.claude/settings.json` (project policy) | Git | Public (committed) |

Memory setup: `autoMemoryDirectory: ".claude/memory"` in
`.claude/settings.local.json`. Both `settings.local.json` and `memory/`
propagate via Syncthing so your working state follows you across machines
without polluting the public repo.

Rule of thumb: **if it's useful to every contributor, it lives in git**
(skills, CLAUDE.md, code). **If it's useful only to you, it lives in the
Syncthing-synced half** (memory, personal settings).

## Cross-References

- `/ov-internals:go` — Source code structure, adding new commands
- `/ov-internals:generate-source` — Understanding generated Containerfiles
- `/ov-build:validate` — Validation rules
- All `/ov:*` skills — Individual command documentation

## When to Use This Skill

Invoke when updating documentation, creating new skills, auditing skill coverage, or deciding where new information belongs (CLAUDE.md vs skill vs memory).
