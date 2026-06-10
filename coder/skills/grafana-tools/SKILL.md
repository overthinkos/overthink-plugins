---
name: grafana-tools
description: |
  Grafana observability CLI tools: mcp-grafana, logcli, promtool, mimirtool, tempo-cli, tanka, grafanactl.
  Use when working with Grafana, Prometheus, Loki, Mimir, Tempo, or observability tooling.
---

# grafana-tools -- Grafana observability CLI suite

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `task:` |

## Usage

```yaml
# charly.yml
my-image:
  layers:
    - grafana-tools
```

## Used In Images

- (none currently enabled)

## Related Layers
- `/charly-coder:devops-tools` — common companion bundle in observability images
- `/charly-coder:kubernetes-layer` — pairs with Grafana for cluster observability
- `/charly-coder:dev-tools` — typically paired in devops images

## Related Commands
- `/charly-build:build` — fetches Grafana CLI binaries via curl during image build
- `/charly-core:shell` — run mcp-grafana, logcli, promtool, etc. interactively

## When to Use This Skill

Use when the user asks about:

- Grafana CLI tools or MCP server
- Loki log queries via logcli
- Prometheus promtool for rule validation
- Mimir or Tempo CLI tools
- Tanka (tk) for Grafana configuration
- grafanactl for Grafana Cloud management
- Observability stack tooling

## Related

- `/charly-image:layer` — layer authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
