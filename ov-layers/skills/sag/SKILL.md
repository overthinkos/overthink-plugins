---
name: sag
description: |
  ElevenLabs text-to-speech CLI.
  Use when working with the sag layer.
---

# sag -- ElevenLabs TTS CLI

## Layer Properties

| Property | Value |
|----------|-------|
| Install files | `layer.yml`, `user.yml` |
| Depends | `golang` |

## Packages

RPM: `alsa-lib-devel`

## Environment

| Variable | Value |
|----------|-------|
| `GOPATH` | `~/go` |
| PATH append | `~/go/bin` |

## Usage

```yaml
# images.yml or layer.yml
layers:
  - sag
```

## Used In Images

- `openclaw-full` (via `openclaw-full` metalayer)
- `openclaw-full-sway` (via `openclaw-full` metalayer)
- `openclaw-full-ml` (via `openclaw-full-ml` metalayer)

## When to Use This Skill

Use when the user asks about:
- ElevenLabs text-to-speech
- TTS in containers
- The `sag` layer
