# Overthink Plugins

Claude Code plugins for the Overthink container image composition system.

## Which Plugins to Install

| Use Case | Plugins |
|----------|---------|
| **Building and running images** | overthink |
| **Contributing to the ov CLI** | overthink + overthink-dev |

## Plugins

### overthink (6 skills)

Skills for composing, building, and running container images with the `ov` CLI.

| Skill | Invocation | Description |
|-------|-----------|-------------|
| layer | `/overthink:layer` | Layer authoring (layer.yml, root.yml, pixi.toml, etc.) |
| image | `/overthink:image` | Image composition (images.yml, defaults, inheritance) |
| build | `/overthink:build` | Building images (ov build, task build:*, caches) |
| run | `/overthink:run` | Runtime operations (ov shell, start/stop, aliases) |
| deploy | `/overthink:deploy` | Deployment (quadlet, bootc, tunnels, encryption) |
| module | `/overthink:module` | Remote layer modules (ov mod) |

### overthink-dev (3 skills, 3 agents, GitHub MCP)

Development tools and enforcement agents for contributors.

**Skills:**

| Skill | Invocation | Description |
|-------|-----------|-------------|
| go | `/overthink-dev:go` | Go CLI development (build, test, code map) |
| generate | `/overthink-dev:generate` | Containerfile generation and debugging |
| validate | `/overthink-dev:validate` | Validation rules and error handling |

**Agents:**

| Agent | Type | Trigger |
|-------|------|---------|
| layer-validator | Blocking | Before editing layer.yml files |
| root-cause-analyzer | Blocking | Any error in output |
| testing-validator | Blocking | Claiming something "works" |

**MCP Server:** GitHub (22 tools for issues, PRs, workflows, repo operations)

## Installation

### Via Team Configuration (settings.json)

```json
{
  "enabledPlugins": {
    "overthink@overthink-plugins": true,
    "overthink-dev@overthink-plugins": true
  },
  "extraKnownMarketplaces": {
    "overthink-plugins": {
      "source": {
        "source": "directory",
        "path": "./plugins"
      }
    }
  }
}
```

## Component Totals

| Component | Count |
|-----------|-------|
| Plugins | 2 |
| Skills | 9 |
| Agents | 3 |
| MCP Servers | 1 (GitHub) |
| MCP Tools | 22 |
