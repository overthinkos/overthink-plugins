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
# images.yml
my-image:
  layers:
    - vscode
```

## Used In Images

- `/overthink-images:bazzite-ai` (disabled)

## When to Use This Skill

Use when the user asks about:

- VS Code installation in containers
- Microsoft code editor setup
- The `code` RPM package or Microsoft repository
