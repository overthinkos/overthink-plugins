---
name: ov-version
description: |
  Show charly CLI version information.
  MUST be invoked before any work involving: charly version command or checking installed charly version.
  Named `ov-version` (not `version`) to disambiguate from Claude Code's built-in `/version` slash command.
---

# charly version -- CLI Version

## Overview

Displays the installed version of the charly CLI binary.

## Quick Reference

| Action | Command | Description |
|--------|---------|-------------|
| Show version | `charly version` | Print charly version string |

## Usage

```bash
charly version
```

## Testing Note

`charly version` writes the CalVer tag to **stdout** via `fmt.Println`.
Declarative tests should match `stdout:` — for example
`candy/ov/candy.yml` uses:

```yaml
- id: ov-version
  command: /usr/local/bin/charly version
  exit_status: 0
  stdout:
    - matches: "[0-9]{4}\\.[0-9]+"
```

`VersionCmd` writes via `fmt.Println` (not Go's builtin `println`,
which bypasses `os.Stderr` and writes directly to fd 2) so the
in-process MCP tool-call path — which captures `os.Stdout` — surfaces
the version correctly. See `/charly-eval:eval` Authoring Gotcha #5 and
`/charly-build:ov-mcp-cmd` "Capture model" for the capture-pipeline detail.

## Cross-References

- `/charly-core:ov-doctor` -- Full host dependency and health check
- `/charly-build:settings` — runtime config where `secret_backend` and other settings live
- `/charly-image:image` — build-mode family that stamps CalVer tags matching this version
- `/charly-eval:eval` — declarative testing framework (Gotcha #5 covers the stdout rule)
- `/charly-build:ov-mcp-cmd` — MCP server section explains why the stream choice matters for the capture pipeline
- `/charly-internals:go` — `ov/version.go` CalVer computation; `main.go` `VersionCmd.Run` using `fmt.Println`
