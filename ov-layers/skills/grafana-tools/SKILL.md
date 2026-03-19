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
| Install files | `root.yml` |

## Usage

```yaml
# images.yml
my-image:
  layers:
    - grafana-tools
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- Grafana CLI tools or MCP server
- Loki log queries via logcli
- Prometheus promtool for rule validation
- Mimir or Tempo CLI tools
- Tanka (tk) for Grafana configuration
- grafanactl for Grafana Cloud management
- Observability stack tooling
