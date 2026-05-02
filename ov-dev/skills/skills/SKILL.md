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

CLAUDE.md R0 (SKILLS FIRST — THE SUPREME RULE) is the authoritative dispatcher. This section mirrors it so skill-authors already inside `/ov-dev:skills` see the same mapping without having to context-switch back to CLAUDE.md. If this table ever drifts from CLAUDE.md R0, **CLAUDE.md R0 wins** — fix this file, never the other way around.

| Trigger | Skills to load BEFORE doing anything |
|---|---|
| `ov rebuild` / `ov vm *` / VM entities | `/ov-advanced:vm` + `/ov-dev:vm-deploy-target` |
| `ov deploy add/del` / pod or container deploys | `/ov-core:deploy` |
| host-target / nested host deploy | `/ov-advanced:local-deploy` + `/ov-dev:local-infra` |
| `ov eval live` / `ov eval cdp/wl/dbus/vnc/mcp/record/spice/libvirt` | `/ov-build:eval` |
| `ov eval k8s <verb>` | `/ov-advanced:eval-k8s` |
| Editing `layer.yml` / layer authoring | `/ov-build:layer` |
| Editing `image.yml` / image composition | `/ov-build:image` |
| `ov image build` / `ov image generate` / Containerfile | `/ov-build:build` + `/ov-build:generate` + `/ov-dev:generate` |
| `ov image validate` / schema error | `/ov-build:validate` |
| Secret management / `ov secrets` / `.kdbx` | `/ov-build:secrets` |
| Schema v4 migration / legacy → new format | `/ov-build:migrate` |
| Hard-cutover concerns / rename sweeps | `/ov-dev:cutover-policy` |
| `disposable: true` semantics | `/ov-dev:disposable` |
| Go source work | `/ov-dev:go` |
| IR / InstallPlan / DeployTarget / OCITarget | `/ov-dev:install-plan` |
| OCI labels / capabilities contract | `/ov-dev:capabilities` |
| VmSpec / libvirt / cloud-init / OVMF | `/ov-dev:vm-spec` + renderer skills |
| Unexpected failure / anomaly | `/ov-dev:root-cause-analyzer` agent |
| "What does layer X do?" | `/ov-layers:<name>` |
| "What's in image X?" | `/ov-images:<name>` |
| Skill authoring / maintenance | `/ov-dev:skills` (this skill) |

If multiple triggers apply, load ALL matching skills in ONE message (parallel `Skill` calls). Full index: `plugins/README.md` (250+ skills).

## When to Update Skills

| Trigger | Action |
|---------|--------|
| Deployment step fails or needs undocumented workaround | Update the relevant `/ov:*` or `/ov-images:*` skill |
| Verification check missing from image skill | Add to the image skill's Verification section |
| Skill's recommended defaults are wrong | Fix in the skill, not CLAUDE.md |
| New feature added to ov CLI | Update `/ov:<cmd>` skill + `/ov-dev:go` source map |
| New layer or image added | Create skill via `ov image new layer` scaffold or manual SKILL.md |
| Bug fix changes behavior | Document the fix in affected skills |
| Cross-skill behavior discovered | Update Cross-References in all affected skills |

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
| Command usage, flags, examples | `/ov:<cmd>` skill |
| Layer properties, packages, ports | `/ov-layers:<name>` skill |
| Image composition, deployment, verification | `/ov-images:<name>` skill |
| Skill disambiguation (which skill to use) | CLAUDE.md (brief table) |
| Detailed operational patterns | Relevant `/ov:*` skill |

## Command skills vs topic skills

Most skills under `plugins/ov/skills/` map 1:1 to a top-level ov command
(e.g. `/ov-build:build` ↔ `ov image build`, `/ov-core:status` ↔ `ov status`).
**Topic skills** are the exception: they don't correspond to a
top-level command but cover a cross-cutting concept surfaced by flags
or layer composition. Today's topic skills:

| Skill | Surfaced via | What it covers |
|---|---|---|
| `/ov-advanced:enc` | `ov config --encrypt`, `ov config mount`, `ov config unmount`, `ov config passwd` | Encrypted-volume (gocryptfs) semantics, keyring resolution, `ov-enc-<image>-<volume>.scope` lifecycle |
| `/ov-advanced:openclaw` | Composing `openclaw-*` layers | OpenClaw AI gateway deployment story |
| `/ov-advanced:sidecar` | `ov config --sidecar tailscale` | Sidecar-container model, pod networking, env-var routing |

When adding a new command, always create a matching command skill. Consider a topic skill when a concept spans multiple commands or layers and the natural home isn't any single command's skill. Keep the frontmatter `description:` explicit about the topic nature (the blocking `Skill:` tool dispatcher matches on description keywords).

## Plugin Structure

| Plugin | Skills | Purpose |
|--------|--------|---------|
| `ov` | 38 | CLI operations ("how do I use X?") — includes `/ov-build:eval` |
| `ov-dev` | 3 + 3 agents | Development ("how does the code work?") |
| `ov-jupyter` | 1 MCP server | Notebook MCP tools |
| `ov-layers` | 161 | Layer reference ("what does layer X contain?") |
| `ov-images` | 41 | Image reference ("what does image X look like?") |

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

- `/ov-dev:go` — Source code structure, adding new commands
- `/ov-dev:generate` — Understanding generated Containerfiles
- `/ov-build:validate` — Validation rules
- All `/ov:*` skills — Individual command documentation

## When to Use This Skill

Invoke when updating documentation, creating new skills, auditing skill coverage, or deciding where new information belongs (CLAUDE.md vs skill vs memory).
