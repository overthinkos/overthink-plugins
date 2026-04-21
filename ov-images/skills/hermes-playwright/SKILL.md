---
name: hermes-playwright
description: |
  Hermes AI agent image with Playwright Chromium browser for web automation.
  Builds on top of the headless hermes image, adding Chromium and system deps.
  MUST be invoked before building, deploying, configuring, or troubleshooting the hermes-playwright image.
---

# hermes-playwright

Hermes AI agent with Playwright Chromium — web scraping, browser automation, and all headless agent capabilities.

## Image Properties

| Property | Value |
|----------|-------|
| Base | fedora |
| Layers | agent-forwarding, hermes, hermes-playwright, dbus, ov |
| Platforms | linux/amd64 |
| Security | shm_size: 1g |
| Registry | ghcr.io/overthinkos |

## Full Layer Stack

Builds on the base `hermes` layer (not the hermes image), adding Playwright Chromium:

1. `fedora` base + `agent-forwarding` + `hermes` + `hermes-playwright` + `dbus` + `ov`
2. `hermes-playwright` -- Playwright npm package + Chromium Headless Shell + system deps

## Quick Start

```bash
ov image build hermes-playwright
ov config hermes-playwright -e OLLAMA_API_KEY=your-key   # or OPENROUTER_API_KEY
ov start hermes-playwright
```

The hermes entrypoint performs single-phase, first-start-only auto-configuration of LLM providers and MCP servers from env vars. See `/ov-images:hermes` for full provider configuration and MCP auto-discovery details.

## Playwright Usage

### From Node.js
```bash
ov shell hermes-playwright -c "NODE_PATH=~/.npm-global/lib/node_modules node -e \"
const { chromium } = require('playwright');
(async () => {
  const browser = await chromium.launch({ headless: true });
  const page = await browser.newPage();
  await page.goto('https://example.com');
  console.log(await page.title());
  await browser.close();
})();
\""
```

### Via npx CLI
```bash
ov shell hermes-playwright -c "npx playwright --version"
```

## Key Layers

- `/ov-layers:hermes` -- core agent (inherited)
- `/ov-layers:hermes-playwright` -- Playwright + Chromium + system deps

## Fedora Compatibility Note

Playwright's `--with-deps` flag does not support Fedora (falls back to Ubuntu's `apt-get`). The `hermes-playwright` layer works around this by:
1. Installing Chromium's system library dependencies via rpm packages in `layer.yml`
2. Installing only the browser binary via `npx playwright install chromium` in `tasks:`

The `PLAYWRIGHT_BROWSERS_PATH=/tmp/.cache/ms-playwright` env var is set automatically.

## Related Images

- `/ov-images:hermes` -- full-featured standalone hermes (no browser, uses cross-container CDP)
- `/ov-images:openclaw-sway-browser` -- alternative: OpenClaw with full desktop + Chrome

## Verification

After `ov start`:
```bash
ov status hermes-playwright               # container running
ov service status hermes-playwright        # all services RUNNING
ov shell hermes-playwright -c "hermes --version"
ov shell hermes-playwright -c "npx playwright --version"
# Full browser launch test:
ov shell hermes-playwright -c "NODE_PATH=~/.npm-global/lib/node_modules node -e \"
const { chromium } = require('playwright');
(async () => {
  const b = await chromium.launch({ headless: true });
  const p = await b.newPage();
  await p.goto('https://example.com');
  console.log('Title:', await p.title());
  await b.close();
  console.log('OK');
})();
\""
```

## When to Use This Skill

**MUST be invoked** when the task involves the hermes-playwright image, Hermes Agent with browser automation, or deploying hermes with Playwright. Invoke this skill BEFORE reading source code or launching Explore agents.

## Related

- `/ov:image` — image family umbrella (`image:` entries in `overthink.yml`, build/validate/inspect/list)
- `/ov:build` — `build.yml` vocabulary (distros, builders, init-systems)
