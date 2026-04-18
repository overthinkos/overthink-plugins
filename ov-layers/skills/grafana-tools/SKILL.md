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
| Install files | `tasks:` |

## Usage

```yaml
# image.yml
my-image:
  layers:
    - grafana-tools
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:devops-tools` — common companion bundle in observability images
- `/ov-layers:kubernetes` — pairs with Grafana for cluster observability
- `/ov-layers:dev-tools` — typically paired in devops images

## Related Commands
- `/ov:build` — fetches Grafana CLI binaries via curl during image build
- `/ov:shell` — run mcp-grafana, logcli, promtool, etc. interactively

## When to Use This Skill

Use when the user asks about:

- Grafana CLI tools or MCP server
- Loki log queries via logcli
- Prometheus promtool for rule validation
- Mimir or Tempo CLI tools
- Tanka (tk) for Grafana configuration
- grafanactl for Grafana Cloud management
- Observability stack tooling
