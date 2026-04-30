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

- `/ov-foundation:bazzite-ai` (disabled)

## Related Layers
- `/ov-selkies:desktop-apps` — desktop tooling sibling
- `/ov-coder:dev-tools` — CLI dev tools companion
- `/ov-coder:typst` — document tooling sibling

## Related Commands
- `/ov-core:shell` — launch vscode inside the container
- `/ov-build:build` — rebuild after layer changes

## When to Use This Skill

Use when the user asks about:

- VS Code installation in containers
- Microsoft code editor setup
- The `code` RPM package or Microsoft repository

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
