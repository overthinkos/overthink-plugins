---
name: vectorchord
description: |
  VectorChord PostgreSQL extension for optimized vector similarity search.
  Use when working with VectorChord, vector indices, or smart search performance.
---

# vectorchord -- VectorChord vector similarity extension for PostgreSQL

## Layer Properties

| Property | Value |
|----------|-------|
| Dependencies | `postgresql` |
| Ports | none |
| Volumes | none |
| Service | none |
| Install files | `tasks:` |

## Environment Variables

| Variable | Value |
|----------|-------|
| `VECTORCHORD_VERSION` | `1.1.1` |
| `POSTGRES_SHARED_PRELOAD_LIBRARIES` | `vchord.so` |

## Packages

- `unzip` (RPM) -- for extracting VectorChord release archives

## Build Process (tasks:)

Downloads VectorChord from GitHub releases (tensorchord/VectorChord) and installs the extension:

1. Detects PostgreSQL major version via `postgres --version`
2. Detects extension directories via `plpgsql.so` anchor and standard Fedora paths
3. Downloads `postgresql-<PG_MAJOR>-vchord_<VERSION>_<ARCH>-linux-gnu.zip`
4. Extracts `vchord.so` to `pkglibdir` and `.control`/`.sql` files to `sharedir/extension/`

Supports both `x86_64` and `aarch64` architectures. PostgreSQL version is detected dynamically.

## How It Works

VectorChord provides the `vchordrq` index type for PostgreSQL, replacing pgvector's `ivfflat` with an optimized approximate nearest-neighbor search algorithm. Key differences:

- **No result cap** -- pgvector's ivfflat has an inherent limit (~40 results); vchordrq does not
- **`shared_preload_libraries`** -- VectorChord's `.so` must be loaded at PostgreSQL startup via the `POSTGRES_SHARED_PRELOAD_LIBRARIES` env var (consumed by the postgresql layer's entrypoint)
- **Extension creation** -- `CREATE EXTENSION vchord CASCADE;` (handled by the immich layer's migration script)

## Version Requirements (Immich v2.7.2)

- pgvector >= 0.7, < 0.9 (provided by postgresql layer)
- VectorChord >= 0.3, < 2.0 (provided by this layer)
- PostgreSQL >= 14, < 19 (Fedora 43 ships PG 18)

## Usage

```yaml
# image.yml -- used alongside postgresql for Immich smart search
my-image:
  layers:
    - postgresql
    - vectorchord
    - immich
```

## Used In Images

- `/ov-immich:immich`
- `/ov-immich:immich-ml`

## Related Layers

- `/ov-foundation:postgresql` -- required dependency (provides PostgreSQL server and pgvector)
- `/ov-immich:immich` -- primary consumer (migration script creates the extension)

## When to Use This Skill

Use when the user asks about:

- VectorChord extension installation or configuration
- Vector similarity search performance
- PostgreSQL shared_preload_libraries configuration
- Immich smart search or face recognition index types
- `vchordrq` index type

## Related

- `/ov-build:layer` — layer authoring reference (`layer.yml` schema, task verbs, service declarations)
- `/ov-build:eval` — declarative testing (`tests:` block, `ov eval image`, `ov test`)
