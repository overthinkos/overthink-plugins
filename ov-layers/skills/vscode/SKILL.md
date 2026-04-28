---
name: vscode
description: |
  Visual Studio Code editor installed from Microsoft's RPM repository.
  Use when working with VS Code installation or configuration in container images.
---

# vscode -- Visual Studio Code editor

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml` |

## Packages

- `code` (RPM, from Microsoft repo) -- Visual Studio Code

## Usage

```yaml
# image.yml
my-image:
  layers:
    - vscode
```

## Used In Images

- `/ov-images:bazzite-ai` (disabled)

## Related Layers
- `/ov-layers:desktop-apps` — desktop tooling sibling
- `/ov-layers:dev-tools` — CLI dev tools companion
- `/ov-layers:typst` — document tooling sibling

## Related Commands
- `/ov:shell` — launch vscode inside the container
- `/ov:build` — rebuild after layer changes

## When to Use This Skill

Use when the user asks about:

- VS Code installation in containers
- Microsoft code editor setup
- The `code` RPM package or Microsoft repository

## Related

- `/ov:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov:test` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
