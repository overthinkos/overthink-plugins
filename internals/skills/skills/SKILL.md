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

CLAUDE.md R0 (SKILLS FIRST — THE SUPREME RULE) is the authoritative dispatcher. This section mirrors it so skill-authors already inside `/charly-internals:skills` see the same mapping without having to context-switch back to CLAUDE.md. If this table ever drifts from CLAUDE.md R0, **CLAUDE.md R0 wins** — fix this file, never the other way around.

| Trigger | Skills to load BEFORE doing anything |
|---|---|
| `charly update` / `charly vm *` / VM entities | `/charly-vm:vm` + `/charly-internals:vm-deploy-target` |
| `charly deploy add/del` / pod or container deploys | `/charly-core:deploy` |
| host-target / nested host deploy | `/charly-local:local-deploy` + `/charly-internals:local-infra` |
| `charly eval live` / `charly eval cdp/wl/dbus/vnc/mcp/record/spice/libvirt` | `/charly-eval:eval` |
| `charly eval k8s <verb>` | `/charly-kubernetes:eval-k8s` |
| Editing `candy.yml` / layer authoring | `/charly-image:layer` |
| Editing `box.yml` / image composition | `/charly-image:image` |
| `charly box build` / `charly box generate` / Containerfile | `/charly-build:build` + `/charly-build:generate` + `/charly-internals:generate-source` |
| `charly box validate` / schema error | `/charly-build:validate` |
| Secret management / `charly secrets` / Secret Service / GPG `.secrets` | `/charly-build:secrets` |
| Schema migration / legacy → latest CalVer | `/charly-build:migrate` |
| Git/`gh` workflow — `feat/` branch, commit, push, ff-merge, tag, worktree, sync, prune, PR, approve | `/charly-internals:git-workflow` |
| `charly box reconcile` / cross-repo `@github` pin alignment | `/charly-build:reconcile` |
| Hard-cutover concerns / rename sweeps | `/charly-internals:cutover-policy` |
| `disposable: true` semantics | `/charly-internals:disposable` |
| Go source work | `/charly-internals:go` |
| IR / InstallPlan / DeployTarget / OCITarget | `/charly-internals:install-plan` |
| OCI labels / capabilities contract | `/charly-internals:capabilities` |
| VmSpec / libvirt / cloud-init / OVMF | `/charly-internals:vm-spec` + renderer skills |
| Unexpected failure / anomaly | `/charly-internals:root-cause-analyzer` agent |
| Engineering-discipline trigger (failure / dup pattern / ad-hoc fix / "out of scope" framing) | `/charly-internals:strict-policy` |
| "What does layer X do?" — pod-specific | `/charly-jupyter:<name>`, `/charly-coder:<name>`, `/charly-selkies:<name>`, `/charly-openclaw:<name>`, `/charly-ollama:<name>`, `/charly-openwebui:<name>`, `/charly-comfyui:<name>`, `/charly-immich:<name>`, `/charly-hermes:<name>`, `/charly-filebrowser:<name>` |
| "What does layer X do?" — base distros / GPU / bootc | `/charly-distros:<name>` (arch, fedora, debian, ubuntu, nvidia, cuda, rocm, bootc-base, …) |
| "What does layer X do?" — language runtime | `/charly-languages:<name>` (python, python-ml, pixi) |
| "What does layer X do?" — infrastructure service | `/charly-infrastructure:<name>` (postgresql, redis, k3s, traefik, supervisord, tailscale, gocryptfs, virtualization, dbus-layer, tmux-layer, …) |
| "What does layer X do?" — CLI utility | `/charly-tools:<name>` (ripgrep, himalaya, whisper, charly, …) |
| Skill authoring / maintenance | `/charly-internals:skills` (this skill) |

If multiple triggers apply, load ALL matching skills in ONE message (parallel `Skill` calls). Full index: `plugins/README.md` (250+ skills).

## When to Update Skills

| Trigger | Action |
|---------|--------|
| Deployment step fails or needs undocumented workaround | Update the relevant `/charly-core:*`, `/charly-build:*`, `/charly-eval:*`, `/charly-automation:*`, kind plugin (`/charly-image:*`, `/charly-vm:*`, `/charly-kubernetes:*`, `/charly-local:*`, `/charly-pod:*`), per-pod plugin (`/charly-jupyter:*`, `/charly-coder:*`, …), or split-foundation plugin (`/charly-distros:*`, `/charly-languages:*`, `/charly-infrastructure:*`, `/charly-tools:*`) |
| Verification check missing from image skill | Add to the image skill's Verification section |
| Skill's recommended defaults are wrong | Fix in the skill, not CLAUDE.md |
| New feature added to charly CLI | Update `/charly-core:<cmd>` or `/charly-build:<cmd>` skill + `/charly-internals:go` source map |
| New layer or image added | Create skill via `charly box new candy` scaffold or manual SKILL.md |
| Bug fix changes behavior | Document the fix in affected skills |
| Cross-skill behavior discovered | Update Cross-References in all affected skills |
| A live bed contradicts a skill's claim (Risk Driven Development found it stale) | Fix the stale skill in the SAME change — RDD keeps the living docs honest; for a high-risk claim the running system is ground truth, not the doc |
| Removed identifier still referenced in skill paragraph (R5 self-test failed) | Update / delete the paragraph in the SAME commit as the removal (R5) |

## When NOT to Update Skills

- **Ephemeral issues** — use conversation context or memory
- **User-specific config** — use Claude Code memory system
- **Bug fixes in charly code** — the fix is in git; document behavioral changes in skills only
- **Anything derivable from code** — skills document *usage*, not implementation details
- **Historical / version-history content** — dated change notes, "renamed from", "previously / formerly / was", completed cutovers, retired / relocated identifiers → `CHANGELOG.md` (repo root), NEVER a skill or CLAUDE.md. Skills describe current behavior in present tense only. When a cutover lands, append its narrative to `CHANGELOG.md` and state the new standing rule forward-looking in the skill, with no history.

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
| Command usage, flags, examples | `/charly-core:<cmd>` or `/charly-build:<cmd>` skill |
| Layer properties, packages, ports | per-pod plugin (`/charly-jupyter:<name>`, `/charly-coder:<name>`, …) or split-foundation plugin (`/charly-distros:*`, `/charly-languages:*`, `/charly-infrastructure:*`, `/charly-tools:*`) for base layers |
| Image composition, deployment, verification | per-pod plugin or `/charly-distros:<name>` / `/charly-infrastructure:<name>` for base images |
| Skill disambiguation (which skill to use) | CLAUDE.md (brief table) |
| Detailed operational patterns | Relevant `/charly-core:*` / `/charly-build:*` / `/charly-eval:*` / `/charly-automation:*` / kind-plugin skill |
| Version history / past changes / renames / cutover narration | `CHANGELOG.md` (repo root) — never CLAUDE.md or a skill |
| Long-term thesis / vision / aspiration ("why & where it's going") | `VISION.md` (repo root) — never restating command usage, architecture, or history |

## Command skills vs topic skills

Most skills under `plugins/charly-core/skills/` and `plugins/charly-build/skills/`
map 1:1 to a top-level charly command (e.g. `/charly-build:build` ↔ `charly box build`,
`/charly-core:charly-status` ↔ `charly status`).
**Topic skills** are the exception: they don't correspond to a
top-level command but cover a cross-cutting concept surfaced by flags
or layer composition. Today's topic skills:

| Skill | Surfaced via | What it covers |
|---|---|---|
| `/charly-automation:enc` | `charly config --encrypt`, `charly config mount`, `charly config unmount`, `charly config passwd` | Encrypted-volume (gocryptfs) semantics, keyring resolution, `charly-enc-<image>-<volume>.scope` lifecycle |
| `/charly-automation:openclaw-deploy` | Composing `openclaw-*` layers | OpenClaw AI gateway deployment story |
| `/charly-automation:sidecar` | `charly config --sidecar tailscale` | Sidecar-container model, pod networking, env-var routing |

When adding a new command, always create a matching command skill. Consider a topic skill when a concept spans multiple commands or layers and the natural home isn't any single command's skill. Keep the frontmatter `description:` explicit about the topic nature (the blocking `Skill:` tool dispatcher matches on description keywords).

## Plugin Structure

Plugins are sorted into four use-case buckets. Directory names live at
`plugins/<name>/` (no `charly-` prefix); plugin.json `name:` fields keep the
`charly-` prefix; every skill is invoked as `/charly-<plugin>:<skill>`.

| Bucket | Plugin | Skills | Purpose |
|---|---|---:|---|
| commands | `charly-core` | 14 | Lifecycle verbs (start/stop/status/logs/shell/ssh/deploy/update/...) |
| commands | `charly-build` | 13 | Build/authoring verbs (build/generate/list/inspect/merge/new/pull/validate/secrets/settings/migrate/reconcile/mcp) |
| commands | `charly-eval` | 9 | `charly eval` orchestrator + live probes (cdp/wl/wl-overlay/dbus/vnc/spice/libvirt/record) |
| commands | `charly-automation` | 6 | tmux verb, host-side helpers (alias/udev), topic flags (enc/sidecar/openclaw-deploy) |
| kind | `charly-image` | 2 | `kind: box` and `kind: candy` schema reference |
| kind | `charly-vm` | 7 | `kind: vm` schema + bootc VM catalog |
| kind | `charly-kubernetes` | 2 | `kind: k8s` schema + cluster probes |
| kind | `charly-local` | 2 | `kind: local` schema + ssh-host deploys |
| kind | `charly-pod` | 1 | `kind: pod` and `kind: deploy` schema (thin pointer) |
| development | `charly-internals` | 16 + 5 agents | Go source / IR / capabilities / vm-spec / renderers / cutover-policy / strict-policy / disposable / git-workflow / agents (the agents/workflows/teams guide) + 5 agents (3 enforcers + 2 eval executors) + github MCP |
| images | `charly-distros` | 34 | Base OS, GPU runtime, bootc, distro builders |
| images | `charly-languages` | 4 | python, python-ml, pixi |
| images | `charly-infrastructure` | 22 | postgres, redis, k3s, traefik, supervisord, tailscale, gocryptfs, virtualization, dbus-layer, tmux-layer, ... |
| images | `charly-tools` | 19 | CLI utilities + charly binary deploy |
| images | `charly-jupyter` | 15 | jupyter image family + jupyter MCP @ 8888 |
| images | `charly-coder` | 31 | coder/dev images + charly MCP @ 18765 |
| images | `charly-selkies` | 45 | selkies-desktop family + chrome-devtools MCP @ 9224 |
| images | `charly-openclaw` | 12 | openclaw AI workstation + chrome-devtools MCP @ 9224 |
| images | `charly-versa` | 9 | versa image — marimo + airflow + OSM analytics + 2 MCP servers |
| images | `charly-ollama` | 2 | ollama LLM-server image |
| images | `charly-openwebui` | 2 | openwebui chat frontend |
| images | `charly-comfyui` | 2 | comfyui image generation |
| images | `charly-immich` | 4 | immich photo management |
| images | `charly-hermes` | 6 | hermes agent image |
| images | `charly-filebrowser` | 2 | filebrowser web file management |

## Agent & signpost conventions

### Agents (`plugins/<plugin>/agents/<name>.md`)

Sub-agents are markdown + YAML frontmatter (`name`, `description`, `tools`,
`model`, …), discovered from a plugin's `agents/` directory (currently only
`charly-internals/agents/`). **Plugin-loaded agents IGNORE the `hooks`,
`mcpServers`, and `permissionMode` frontmatter fields** — keep those out of
plugin agents (use `.claude/agents/` or `settings.json` if you genuinely
need them). The charly roster splits into **enforcers** (root-cause-analyzer,
layer-validator, testing-validator — gate claims) and **executors**
(eval-bed-runner, deploy-verifier — drive `charly eval` and return verbatim
proof). Full story: `/charly-internals:agents`. Dynamic workflows are NOT plugin
content — they live in the superproject's `.claude/workflows/*.js`.

### Per-directory CLAUDE.md signposts (hybrid)

The repo-root `CLAUDE.md` is the single canonical R0–R10 rule-set.
Per-directory `CLAUDE.md` files (`charly/`, `candy/`, `plugins/`, and each
`image/<distro>` submodule) are THIN signposts only: they name the skills to
load for that area and point back to root. They MUST NOT restate any rule —
duplication drifts (the hooks and an earlier layer-validator both drifted
exactly this way). Subagents and teammates load the full `CLAUDE.md`
hierarchy from their working directory, so a signpost reaches a worker scoped
to that subtree without bloating root.

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

- `/charly-internals:go` — Source code structure, adding new commands
- `/charly-internals:generate-source` — Understanding generated Containerfiles
- `/charly-internals:agents` — Sub-agents, dynamic workflows, agent teams; how they drive the `charly eval` beds; the hooks doctrine; the signpost convention
- `/charly-build:validate` — Validation rules
- All `/charly:*` skills — Individual command documentation

## When to Use This Skill

Invoke when updating documentation, creating new skills, auditing skill coverage, or deciding where new information belongs (CLAUDE.md vs skill vs memory).
