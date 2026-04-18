---
name: version
description: |
  Show ov CLI version information.
  MUST be invoked before any work involving: ov version command or checking installed ov version.
---

# ov version -- CLI Version

## Overview

Displays the installed version of the ov CLI binary.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Show version | `ov version` | Print ov version string |

## Usage

```bash
ov version
```

## Testing Note

`ov version` writes to **stderr**, not stdout — declarative tests must
use `stderr:` matcher. See `/ov:test` Authoring Gotcha #5.

## Cross-References

- `/ov:doctor` -- Full host dependency and health check
- `/ov:settings` — runtime config where `secret_backend` and other settings live
- `/ov:image` — build-mode family that stamps CalVer tags matching this version
- `/ov:test` — declarative testing framework (stderr vs stdout gotcha)
- `/ov-dev:go` — `ov/version.go` CalVer computation
