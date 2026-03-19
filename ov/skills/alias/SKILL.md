---
name: alias
description: |
  Command aliases: ov alias add/remove/list/install/uninstall commands.
  Use when creating host command wrappers that run inside containers.
---

# Alias - Command Aliases

## Overview

`ov alias` creates distrobox-style wrapper scripts in `~/.local/bin/` that transparently run commands inside containers. Typing the alias name on the host executes the command inside the container via `ov shell`.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Install defaults | `ov alias install <image>` | Create wrappers from layer/image config |
| Uninstall all | `ov alias uninstall <image>` | Remove all wrappers for an image |
| Add manually | `ov alias add <name> <image> [command]` | Create a single wrapper |
| Remove one | `ov alias remove <name>` | Remove a single wrapper |
| List all | `ov alias list` | Show all installed aliases |

## How It Works

```bash
ov alias install openclaw     # Creates ~/.local/bin/openclaw
openclaw                      # Runs inside container transparently
ov alias uninstall openclaw   # Removes wrapper
```

### Wrapper Script Format

`ov alias add` / `ov alias install` writes shell scripts to `~/.local/bin/`:

```sh
#!/bin/sh
# ov-alias
# image: openclaw
# command: openclaw
_ov_q(){ printf "'"; printf '%s' "$1" | sed "s/'/'\\\\''/g"; printf "' "; }
c="openclaw"; for a in "$@"; do c="$c $(_ov_q "$a")"; done
exec ov shell openclaw -c "$c"
```

Key details:
- The `# ov-alias` marker enables safe list/delete scanning
- `ov alias remove` verifies the marker before deleting
- The `_ov_q()` helper properly single-quotes each argument (POSIX sh compatible)
- Arguments are forwarded to `ov shell <image> -c "<command> <args>"`

## Declaration

### In layer.yml

Layer aliases require both `name` and `command`:

```yaml
aliases:
  - name: ollama
    command: ollama
  - name: ollama-run
    command: "ollama run"
```

### In images.yml

Image-level aliases default `command` to `name` if omitted. Image-level aliases override layer aliases with the same name.

```yaml
images:
  my-app:
    aliases:
      - name: myapp          # command defaults to "myapp"
      - name: myctl
        command: "myapp ctl"
```

### Collection

`CollectImageAliases()` gathers aliases from the image's own layers (in dependency order) plus image-level config. **No base chain traversal** -- aliases are leaf-image specific (unlike volumes).

Source: `ov/alias.go`, `ov/layers.go` (`AliasYAML`, `HasAliases`, `Aliases()`).

## Commands

### Install All Aliases for an Image

```bash
ov alias install openclaw
# Created alias: openclaw -> openclaw (in openclaw)
```

Installs all aliases declared in the image's layers and image config.

### Uninstall All Aliases for an Image

```bash
ov alias uninstall openclaw
# Removed alias: openclaw
```

Removes all wrapper scripts that reference the image.

### Add a Single Alias

```bash
ov alias add mycommand my-image                   # command = mycommand
ov alias add mycommand my-image "custom command"   # explicit command
```

### Remove a Single Alias

```bash
ov alias remove mycommand
```

Only removes if the file contains the `# ov-alias` marker.

### List All Aliases

```bash
ov alias list
# NAME         IMAGE        COMMAND
# openclaw     openclaw     openclaw
# ollama       ollama       ollama
```

Scans `~/.local/bin/` for files with the `# ov-alias` marker.

## Validation Rules

- Layer aliases require both `name` and `command`
- Alias names must match `^[a-zA-Z0-9][a-zA-Z0-9._-]*$`
- Duplicate alias names within an image are errors
- Image-level alias `command` defaults to `name` if omitted

## Cross-References

- `/ov:shell` -- How aliases execute commands via `ov shell -c`
- `/ov:layer` -- Declaring aliases in layer.yml
- `/ov:image` -- Image-level alias overrides

## When to Use This Skill

Use when the user asks about:

- `ov alias` commands (add, remove, list, install, uninstall)
- Creating host command wrappers
- Distrobox-style command forwarding
- "How do I run a container command from the host?"
