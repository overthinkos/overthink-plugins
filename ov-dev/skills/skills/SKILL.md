---
name: skills
description: |
  Skill maintenance guidelines: when and how to update skills, CLAUDE.md, and README.md.
  Use when updating documentation, feeding back operational insights, or auditing skill coverage.
---

# Skill Maintenance Guidelines

## Overview

Skills are living documents at `plugins/<plugin>/skills/<name>/SKILL.md`. They are the primary knowledge source for Claude Code — always invoked before codebase exploration. This skill covers when and how to update them.

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

## Plugin Structure

| Plugin | Skills | Purpose |
|--------|--------|---------|
| `ov` | 37 | CLI operations ("how do I use X?") |
| `ov-dev` | 3 + 3 agents | Development ("how does the code work?") |
| `ov-jupyter` | 1 MCP server | Notebook MCP tools |
| `ov-layers` | 161 | Layer reference ("what does layer X contain?") |
| `ov-images` | 41 | Image reference ("what does image X look like?") |

## Cross-References

- `/ov-dev:go` — Source code structure, adding new commands
- `/ov-dev:generate` — Understanding generated Containerfiles
- `/ov:validate` — Validation rules
- All `/ov:*` skills — Individual command documentation

## When to Use This Skill

Invoke when updating documentation, creating new skills, auditing skill coverage, or deciding where new information belongs (CLAUDE.md vs skill vs memory).
