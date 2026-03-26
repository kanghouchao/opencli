# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What is OpenCLI

OpenCLI turns any website, Electron app, or local CLI tool into a command-line interface. It connects to your running Chrome browser via a Browser Bridge extension + WebSocket daemon, reusing your existing login sessions. It also controls Electron desktop apps via CDP.

## Commands

```bash
# Development
npm run dev                       # Run via tsx (no build needed)
npm run build                     # tsc + copy YAML files + build manifest
npm run lint                      # Type check only (tsc --noEmit)
npx tsc --noEmit                  # Same as lint

# Testing
npm test                          # Unit tests (src/**/*.test.ts, excluding clis/)
npm run test:adapter              # Adapter tests (bilibili, zhihu, v2ex)
npx vitest run tests/e2e/         # E2E tests (real CLI execution)
npx vitest run tests/smoke/       # Smoke tests (API health + registry integrity)
npx vitest run                    # All tests
npx vitest run src/output.test.ts # Single test file
npx vitest src/                   # Watch mode

# Extended browser E2E (opt-in, requires Chrome + extension)
OPENCLI_E2E=1 npx vitest run

# Docs
npm run docs:dev
```

After editing source, `npm run build` is required before running E2E or smoke tests (they test `dist/main.js`).

## Architecture

**Dual-Engine Architecture**: YAML declarative pipelines and TypeScript adapters share the same registry and execution path.

```
src/main.ts              → Entry point
src/cli.ts               → Commander.js setup + built-in commands (list, explore, etc.)
src/commanderAdapter.ts  → Bridges Registry commands → Commander subcommands
src/registry.ts          → Central command registry; all adapters call cli() here
src/discovery.ts         → Discovers .yaml/.ts adapters, builds manifest
src/execution.ts         → Arg validation, lazy-loads adapter modules, runs handlers
src/browser.ts           → Chrome connection via Browser Bridge WebSocket
src/pipeline/            → YAML pipeline engine (fetch, map, limit, filter, download steps)
src/output.ts            → Unified output formatting (table/json/yaml/md/csv)
src/doctor.ts            → Self-diagnostic tool
src/clis/<site>/         → All site adapters (YAML or TS files)
extension/               → Chrome Browser Bridge extension
tests/e2e/               → E2E tests using runCli() subprocess helpers
tests/smoke/             → API health and registry integrity checks
```

### Authentication Strategies (`src/registry.ts` `Strategy` enum)

| Strategy | When to use |
|----------|-------------|
| `PUBLIC` | Public APIs — no auth |
| `COOKIE` | Browser session cookies via Browser Bridge |
| `HEADER` | Custom auth headers / tokens |
| `INTERCEPT` | Network request interception (e.g. Twitter GraphQL) |
| `UI` | DOM interaction via accessibility snapshot (desktop apps) |

## Adding a New Adapter

**Use YAML** for simple data-fetching commands. **Use TypeScript** for browser-side logic, multi-step flows, or DOM interaction.

### YAML (`src/clis/<site>/<command>.yaml`)

```yaml
site: mysite
name: trending
description: Trending posts on MySite
domain: www.mysite.com
strategy: public   # public | cookie | header
browser: false     # true if browser session is needed

args:
  query:
    positional: true   # primary arg — typed directly (opencli mysite search "rust")
    type: str
    required: true
  limit:
    type: int
    default: 20

pipeline:
  - fetch:
      url: https://api.mysite.com/trending?q=${{ args.query }}
  - map:
      rank: ${{ index + 1 }}
      title: ${{ item.title }}
  - limit: ${{ args.limit }}

columns: [rank, title, url]
```

Reference: `src/clis/hackernews/top.yaml`

### TypeScript (`src/clis/<site>/<command>.ts`)

```typescript
import { cli, Strategy } from '../../registry.js';
import { EmptyResultError, CommandExecutionError } from '../../errors.js';

cli({
  site: 'mysite',
  name: 'search',
  description: 'Search MySite',
  domain: 'www.mysite.com',
  strategy: Strategy.COOKIE,
  args: [
    { name: 'query', positional: true, required: true, help: 'Search query' },
    { name: 'limit', type: 'int', default: 10, help: 'Max results' },
  ],
  columns: ['title', 'url'],

  func: async (page, kwargs) => {
    const { query, limit = 10 } = kwargs;
    const data = await page.evaluate(`
      fetch('/api/search?q=${encodeURIComponent(String(query))}', { credentials: 'include' })
        .then(r => r.json())
    `);
    if (!data?.results?.length) throw new EmptyResultError('mysite search', 'Try a different keyword');
    return data.results.slice(0, Number(limit)).map((item: any) => ({ title: item.title, url: item.url }));
  },
});
```

Throw typed errors from `src/errors.ts`: `AuthRequiredError`, `EmptyResultError`, `CommandExecutionError`, `TimeoutError`, `ArgumentError`. Never throw raw `Error` for expected adapter failures.

### Arg Design Convention

- **Positional** (`positional: true`): the primary required subject — query, id, url, username
- **Named flags** (`--flag`): optional configuration — `--limit`, `--sort`, `--format`, `--output`

### AI-Assisted Adapter Development

For generating adapters with browser exploration:
- **Quick (single command)**: follow [CLI-ONESHOT.md](./CLI-ONESHOT.md) — open the URL in a browser, capture network requests, reproduce the API with `fetch`, write adapter.
- **Full site**: follow [CLI-EXPLORER.md](./CLI-EXPLORER.md) — multi-step workflow including auth strategy detection.

```bash
opencli explore https://example.com --site mysite   # Discover APIs
opencli synthesize mysite                            # Generate YAML from explore artifacts
opencli generate https://example.com --goal "hot"   # One-shot: explore → synthesize → register
```

### Validate & Test New Adapters

```bash
opencli validate                                         # YAML schema check
opencli <site> <command> --limit 3 -f json               # Manual test
opencli <site> <command> -v                              # Verbose debug
```

Add tests:
- `browser: false` public API → `tests/e2e/public-commands.test.ts`
- `browser: true` public data → `tests/e2e/browser-public.test.ts`
- `browser: true` requires login → `tests/e2e/browser-auth.test.ts`

## Code Conventions

- **ES Modules** — use `.js` extensions in all imports (TypeScript output convention)
- **TypeScript strict mode** — avoid `any`
- **No default exports** — use named exports
- **Naming**: `kebab-case` files, `camelCase` variables/functions, `PascalCase` types/classes
- **Conventional Commits**: `feat(twitter): add thread command`, `fix(browser): handle CDP timeout`

## Test Architecture

| Layer | Location | Run with | Purpose |
|-------|----------|----------|---------|
| Unit | `src/**/*.test.ts` (excl. clis/) | `npm test` | Core modules, pipeline, output |
| Adapter | `src/clis/{bilibili,zhihu,v2ex}/**/*.test.ts` | `npm run test:adapter` | Site-specific logic |
| E2E | `tests/e2e/` | `npx vitest run tests/e2e/` | Real CLI subprocess execution |
| Smoke | `tests/smoke/` | `npx vitest run tests/smoke/` | External API health + registry |

E2E browser tests use `tryBrowserCommand()` with warn+pass for geo-restricted or login-required sites to avoid flaky CI failures.
