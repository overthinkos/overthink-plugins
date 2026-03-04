---
name: module
description: |
  Remote layer modules: Go-module-style dependency management for layers.
  Use when working with layers.mod, layers.lock, or referencing layers
  from external git repositories.
---

# Module - Remote Layer Modules

## Overview

Go-module-style dependency management for layers. Any git repository with a `layers/` directory can serve as a layer module. Remote layers are referenced by fully qualified paths in `images.yml` and cached locally.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Initialize | `ov mod init` | Create layers.mod from git remote |
| Add module | `ov mod get <module>@<version>` | Add/update a require entry |
| Remove module | `ov mod remove <module>` | Remove from layers.mod and lock |
| Download all | `ov mod download` | Download all required modules to cache |
| Tidy | `ov mod tidy` | Remove unused requires, add missing |
| Verify | `ov mod verify` | Check cached modules against lock hashes |
| Update | `ov mod update [module]` | Update to latest version |
| List | `ov mod list` | List modules with versions and layers |

## Layer Referencing

In `images.yml`, remote layers use fully qualified paths. Local layers remain short names:

```yaml
images:
  my-app:
    layers:
      - pixi                                        # local: layers/pixi/
      - github.com/overthinkos/ml-layers/cuda        # remote: module/layer
```

## Module Manifest (`layers.mod`)

Project root, YAML format:

```yaml
module: github.com/overthinkos/overthink
require:
  - module: github.com/overthinkos/ml-layers
    version: v1.0.0
replace:
  - module: github.com/overthinkos/ml-layers
    path: ../ml-layers    # local development override
```

## Lock File (`layers.lock`)

Generated, checked into git. Records resolved state:

```yaml
# layers.lock (generated -- do not edit)
modules:
  - module: github.com/overthinkos/ml-layers
    version: v1.0.0
    commit: a1b2c3d4e5f6...
    hash: sha256:abc123...
    layers: [cuda, python-ml]
```

## Remote Module Structure

Any git repo with `layers/` directory and optional `module.yml`:

```
module.yml              # declares canonical module path
layers/
  cuda/
    layer.yml
    root.yml
```

## Cache

Location: `~/.cache/ov/mod/` (override: `$OV_MODULE_CACHE`). Downloaded via `git clone --depth 1`. Read-only after download.

## Build Context

Remote layers live outside the Docker build context. During `ov generate`, symlinks are created:

```
.build/_layers/cuda -> ~/.cache/ov/mod/github.com/.../layers/cuda
```

## Cross-Module Dependencies

Within a module, `depends` uses short names. Cross-module uses full paths:

```yaml
# In github.com/overthinkos/ml-layers/layers/python-ml/layer.yml
depends:
  - cuda                                    # same module
  - github.com/overthinkos/overthink/pixi   # cross-module
```

## Name Conflicts

- Local layers always shadow remote layers with same short name (note emitted)
- Two remote modules exporting the same layer name is a validation error

## Common Workflows

### Add a Remote Module

```bash
ov mod init                                     # First time
ov mod get github.com/overthinkos/ml-layers@v1.0.0
# Now use in images.yml:
# layers:
#   - github.com/overthinkos/ml-layers/cuda
```

### Local Development Override

Add a `replace` entry in `layers.mod`:

```yaml
replace:
  - module: github.com/overthinkos/ml-layers
    path: ../ml-layers
```

### Update Dependencies

```bash
ov mod update                    # Update all modules
ov mod update github.com/...     # Update specific module
ov mod tidy                      # Remove unused, add missing
```

## Cross-References

- `/overthink:layer` -- Layer definitions
- `/overthink:image` -- Using remote layers in images.yml

## When to Use This Skill

Use when the user asks about:

- Remote layer dependencies
- `layers.mod` or `layers.lock` files
- `ov mod` commands (init, get, tidy, update, verify)
- Sharing layers between projects
- "How do I use layers from another repo?"
