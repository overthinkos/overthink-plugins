---
name: module
description: |
  Remote layer modules: Go-module-style dependency management for layers.
  Use when working with inline @version refs, layers.lock, or referencing
  layers from external git repositories.
---

# Module - Remote Layer Modules

## Overview

Go-module-style dependency management for layers. Any git repository with a `layers/` directory can serve as a layer module. Remote layers are referenced by fully qualified paths with inline `@version` in `images.yml` and `layer.yml`, and cached locally.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Add module | `ov mod get <module>@<version>` | Download module, update layers.lock |
| Download all | `ov mod download` | Download all modules from inline @version refs |
| Tidy | `ov mod tidy` | Remove unused lock entries |
| Verify | `ov mod verify` | Check cached modules against lock hashes |
| Update | `ov mod update [module]` | Update to latest version |
| List | `ov mod list` | List modules with versions and layers |

## Layer Referencing

Remote layers use fully qualified paths with `@version` suffix:

```yaml
# images.yml
images:
  my-app:
    layers:
      - pixi                                                # local: layers/pixi/
      - github.com/overthinkos/ml-layers/cuda@v1.0.0        # remote: module/layer@version
```

```yaml
# layer.yml
depends:
  - github.com/overthinkos/ml-layers/cuda@v1.0.0           # remote dependency
  - supervisord                                              # local dependency
```

Version info is inline in the ref — no separate manifest file needed.

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

Within a module, `depends` uses short names. Cross-module uses full paths with version:

```yaml
# In github.com/overthinkos/ml-layers/layers/python-ml/layer.yml
depends:
  - cuda                                                  # same module
  - github.com/overthinkos/overthink/pixi@main            # cross-module
```

## Name Conflicts

- Local layers always shadow remote layers with same short name (note emitted)
- Two remote modules exporting the same layer name is a validation error
- Same module referenced with different versions is a validation error

## Common Workflows

### Add a Remote Module

```bash
ov mod get github.com/overthinkos/ml-layers@v1.0.0
# Now use in images.yml or layer.yml:
# layers:
#   - github.com/overthinkos/ml-layers/cuda@v1.0.0
```

### Update Dependencies

```bash
ov mod update                    # Update all modules
ov mod update github.com/...     # Update specific module
ov mod tidy                      # Remove unused lock entries
```

## Cross-References

- `/overthink:layer` -- Layer definitions
- `/overthink:image` -- Using remote layers in images.yml

## When to Use This Skill

Use when the user asks about:

- Remote layer dependencies
- `layers.lock` files or inline `@version` refs
- `ov mod` commands (get, tidy, update, verify)
- Sharing layers between projects
- "How do I use layers from another repo?"
