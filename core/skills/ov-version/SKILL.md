---
name: ov-version
description: |
  Show ov CLI version information.
  MUST be invoked before any work involving: ov version command or checking installed ov version.
  Renamed from `version` to `ov-version` to disambiguate from Claude Code's built-in `/version` slash command.
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

`ov version` writes the CalVer tag to **stdout** via `fmt.Println`.
Declarative tests should match `stdout:` — for example
`layers/ov/layer.yml` uses:

```yaml
- id: ov-version
  command: /usr/local/bin/ov version
  exit_status: 0
  stdout:
    - matches: "[0-9]{4}\\.[0-9]+"
```

Historical note: `VersionCmd` originally used Go's builtin
`println(version)` which routes to stderr (the builtin bypasses
`os.Stderr` and writes directly to fd 2). The move to `fmt.Println`
landed with the MCP server work so the in-process tool-call path —
which captures `os.Stdout` — can surface the version correctly. See
`/ov-eval:eval` Authoring Gotcha #5 and `/ov-build:ov-mcp-cmd` "Capture model" for the
full story.

## Cross-References

- `/ov-core:ov-doctor` -- Full host dependency and health check
- `/ov-build:settings` — runtime config where `secret_backend` and other settings live
- `/ov-image:image` — build-mode family that stamps CalVer tags matching this version
- `/ov-eval:eval` — declarative testing framework (Gotcha #5 covers the stdout rule)
- `/ov-build:ov-mcp-cmd` — MCP server section explains why the stream choice matters for the capture pipeline
- `/ov-internals:go` — `ov/version.go` CalVer computation; `main.go` `VersionCmd.Run` using `fmt.Println`
