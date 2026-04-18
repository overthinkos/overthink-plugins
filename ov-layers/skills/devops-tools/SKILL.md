---
name: devops-tools
description: |
  DevOps CLI tools: AWS CLI, Scaleway, kubectx/kubens, OpenTofu, wrangler, bind-utils, jq, rsync.
  Use when working with cloud infrastructure, DNS lookups, infrastructure-as-code, or DevOps tooling.
---

# devops-tools -- cloud and infrastructure CLI tools

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `nodejs` |
| Install files | `tasks:`, `package.json` |

## Packages

- `bind-utils` (RPM) -- DNS lookup tools (dig, nslookup)
- `jq` (RPM) -- JSON processor
- `rsync` (RPM) -- file synchronization
- `wrangler` (npm) -- Cloudflare Workers CLI

## Usage

```yaml
# images.yml
my-image:
  layers:
    - devops-tools
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers

- `/ov-layers:nodejs` -- Node.js dependency for npm packages

## When to Use This Skill

Use when the user asks about:

- AWS CLI, Scaleway CLI, or cloud provider tools
- kubectx or kubens for Kubernetes context switching
- OpenTofu or infrastructure-as-code
- Cloudflare Wrangler
- DNS tools (dig, nslookup) or jq/rsync in containers
