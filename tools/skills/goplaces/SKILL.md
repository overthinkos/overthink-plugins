---
name: goplaces
description: |
  Google Places API CLI for location search.
  Use when working with the goplaces candy.
---

# goplaces -- Google Places API CLI

## Candy Properties

| Property | Value |
|----------|-------|
| Install files | `charly.yml`, `task:` |
| Depends | `golang` |

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# box or candy charly.yml
candy:
  - goplaces
```

## Used In Boxes

- `openclaw-full` (via `openclaw-full` metalayer)

## Related Candies
- `/charly-coder:golang` — required Go toolchain dependency
- `/charly-openclaw:openclaw-full` — metalayer that includes goplaces
- `/charly-tools:gogcli` — sibling Google API CLI in openclaw-full

## Related Commands
- `/charly-build:secrets` — provision Google Places API key for the CLI
- `/charly-core:shell` — run goplaces interactively inside a container
- `/charly-build:build` — compiles goplaces via the Go builder during image build

## When to Use This Skill

Use when the user asks about:
- Google Places API access
- Location search tools
- The `goplaces` candy

## Related

- `/charly-image:layer` — candy authoring reference (`charly.yml` schema, task verbs, service declarations)
- `/charly-eval:eval` — declarative testing (`eval:` block, `charly eval box`, `charly eval live`)
