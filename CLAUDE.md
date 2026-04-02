# CLAUDE.md

Guidelines and context for AI assistants working on this codebase.

## Project Overview

**moltbot-clipcash** is a Cloudflare Worker that runs [Moltbot](https://molt.bot/) (a personal AI assistant) inside a Cloudflare Sandbox container. It provides:

- Reverse proxy to the Moltbot gateway (HTTP + WebSocket)
- React admin UI at `/_admin/` for device management and storage control
- REST API at `/api/admin/*` for device pairing and gateway management
- Debug endpoints at `/debug/*` for troubleshooting (requires `DEBUG_ROUTES=true`)
- Browser automation via Chrome DevTools Protocol at `/cdp`
- Persistent storage via R2 bucket, synced every 5 minutes

**Important naming:** The Moltbot CLI tool is still named `clawdbot` (upstream hasn't renamed yet). Internal config paths and CLI commands use `clawdbot`.

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Runtime | Cloudflare Workers + Sandbox Containers |
| Backend framework | [Hono](https://hono.dev/) v4 |
| Frontend | React 19 + Vite |
| Language | TypeScript (strict mode) |
| Unit tests | Vitest |
| E2E tests | Playwright |
| Build/deploy | Wrangler 4 |
| Storage | Cloudflare R2 (mounted via s3fs) |
| Auth | Cloudflare Access (JWT) + device pairing |

## Project Structure

```
src/
├── index.ts              # Main Hono app entry point; route mounting, middleware chain
├── types.ts              # TypeScript interfaces (MoltbotEnv, AppEnv, etc.)
├── config.ts             # Constants: ports, timeouts, paths
├── test-utils.ts         # Shared Vitest mocking utilities
├── auth/
│   ├── jwt.ts            # CF Access JWT decode and validation
│   ├── jwks.ts           # JWKS fetching and caching
│   └── middleware.ts     # Hono auth middleware (createAccessMiddleware)
├── gateway/
│   ├── process.ts        # Moltbot lifecycle: find existing or start new gateway
│   ├── env.ts            # Build container env vars from Worker secrets
│   ├── r2.ts             # Mount R2 bucket via s3fs
│   ├── sync.ts           # R2 backup (syncToR2) and restore (restoreFromR2)
│   └── utils.ts          # waitForProcess() helper
├── routes/
│   ├── index.ts          # Route barrel exports
│   ├── public.ts         # Unauthenticated routes: health, status, logos, admin assets
│   ├── api.ts            # Admin API: devices, gateway restart, storage
│   ├── admin-ui.ts       # Serve built React admin UI from /_admin/
│   ├── debug.ts          # Debug routes (processes, logs, version)
│   └── cdp.ts            # CDP WebSocket proxy for browser automation
├── client/               # React admin UI (built by Vite, served from /_admin/)
│   ├── main.tsx          # React entry point
│   ├── App.tsx           # Root component with header
│   ├── api.ts            # API client with TypeScript types
│   └── pages/
│       └── AdminPage.tsx # Device pairing, storage status, restart controls
└── utils/
    └── logging.ts        # Logging with sensitive param redaction

Dockerfile               # Container: cloudflare/sandbox base + Node 22 + clawdbot
start-moltbot.sh         # Container startup: configure from env vars, launch gateway
moltbot.json.template    # Default Moltbot config template
wrangler.jsonc           # Worker + Container + R2 + Durable Objects config
skills/
└── cloudflare-browser/  # CDP browser automation skill pre-installed in container
test/
└── e2e/                 # Playwright end-to-end tests
```

## Development Commands

```bash
npm install              # Install dependencies
npm run start            # wrangler dev (local worker, HTTP only — WS needs full deploy)
npm run dev              # Vite dev server (client UI only)
npm run build            # Build worker + client
npm run deploy           # Build and deploy to Cloudflare
npm run typecheck        # TypeScript type check (no emit)
npm test                 # Run all Vitest unit tests once
npm run test:watch       # Run tests in watch mode
npm run test:coverage    # Run tests with V8 coverage + HTML report
npx wrangler tail        # Stream live Worker logs
npx wrangler secret list # List configured secrets
```

## Local Development Setup

```bash
npm install
cp .dev.vars.example .dev.vars
# Edit .dev.vars — at minimum set ANTHROPIC_API_KEY
npm run start
```

Minimal `.dev.vars` for local testing:

```
ANTHROPIC_API_KEY=sk-ant-...
MOLTBOT_GATEWAY_TOKEN=dev-token-change-in-prod
DEV_MODE=true          # Skips CF Access auth + device pairing
DEBUG_ROUTES=true      # Enables /debug/* endpoints
```

**WebSocket limitation:** `wrangler dev` has issues proxying WebSocket connections through the sandbox. HTTP requests work; deploy to Cloudflare for full WebSocket functionality.

## Architecture

```
Browser / Chat Clients
       │
       ▼
┌─────────────────────────────────────┐
│     Cloudflare Worker (index.ts)    │
│  Middleware chain:                  │
│  1. Request logging                 │
│  2. Sandbox init (Durable Objects)  │
│  3. Public routes                   │
│  4. CDP routes (CDP_SECRET auth)    │
│  5. CF Access JWT auth              │
│  6. Admin API + UI + debug routes   │
│  7. Catch-all proxy → Moltbot       │
└──────────────┬──────────────────────┘
               │ HTTP + WebSocket
               ▼
┌─────────────────────────────────────┐
│  Cloudflare Sandbox Container       │
│  ┌───────────────────────────────┐  │
│  │  Moltbot Gateway (:18789)     │  │
│  │  - Control UI                 │  │
│  │  - WebSocket RPC              │  │
│  │  - Agent runtime              │  │
│  └───────────────────────────────┘  │
└─────────────────────────────────────┘
               │
               ▼
     Cloudflare R2 Bucket
     (mounted at /data/moltbot,
      synced every 5 minutes via cron)
```

## API Endpoints

| Method | Path | Auth | Purpose |
|--------|------|------|---------|
| GET | `/sandbox-health` | None | Health check |
| GET | `/api/status` | None | Gateway status |
| GET | `/logo.png`, `/logo-small.png` | None | Static assets |
| GET | `/_admin/assets/*` | None | Admin UI static files |
| GET | `/api/admin/devices` | CF Access | List pending/paired devices |
| POST | `/api/admin/devices/:id/approve` | CF Access | Approve single device |
| POST | `/api/admin/devices/approve-all` | CF Access | Approve all pending |
| POST | `/api/admin/gateway/restart` | CF Access | Restart moltbot gateway |
| GET | `/api/admin/storage` | CF Access | R2 storage status |
| POST | `/api/admin/storage/sync` | CF Access | Manually trigger R2 backup |
| GET | `/debug/processes` | CF Access + `DEBUG_ROUTES` | List container processes |
| GET | `/debug/logs?id=<pid>` | CF Access + `DEBUG_ROUTES` | Get process logs |
| GET | `/debug/version` | CF Access + `DEBUG_ROUTES` | Container/moltbot version |
| WS | `/cdp?secret=<CDP_SECRET>` | `CDP_SECRET` param | Browser automation |
| `*` | `/*` | CF Access | Proxy to Moltbot gateway |

## Environment Variables

### Required (Production)

| Variable | Purpose |
|----------|---------|
| `ANTHROPIC_API_KEY` | Anthropic API key for the AI model |
| `MOLTBOT_GATEWAY_TOKEN` | Token securing the Moltbot gateway (`openssl rand -hex 32`) |
| `CF_ACCESS_TEAM_DOMAIN` | Cloudflare Access team domain (e.g., `team.cloudflareaccess.com`) |
| `CF_ACCESS_AUD` | CF Access application audience tag |

### Optional

| Variable | Purpose |
|----------|---------|
| `AI_GATEWAY_API_KEY` | Provider API key via AI Gateway (overrides `ANTHROPIC_API_KEY`) |
| `AI_GATEWAY_BASE_URL` | AI Gateway endpoint URL |
| `TELEGRAM_BOT_TOKEN` | Telegram bot token |
| `DISCORD_BOT_TOKEN` | Discord bot token |
| `SLACK_BOT_TOKEN` | Slack bot token |
| `SLACK_APP_TOKEN` | Slack app-level token |
| `R2_ACCESS_KEY_ID` | R2 credentials for persistent storage |
| `R2_SECRET_ACCESS_KEY` | R2 credentials for persistent storage |
| `CF_ACCOUNT_ID` | Cloudflare account ID (for R2 mounting) |
| `CDP_SECRET` | Shared secret for CDP WebSocket authentication |
| `WORKER_URL` | Public Worker URL (used by skills) |
| `SANDBOX_SLEEP_AFTER` | Container idle timeout (`never`, `10m`, `1h`, etc.) |

### Development Only

| Variable | Behavior |
|----------|---------|
| `DEV_MODE=true` | Skips CF Access auth AND device pairing (maps to `CLAWDBOT_DEV_MODE` in container) |
| `E2E_TEST_MODE=true` | Skips CF Access but keeps device pairing requirement |
| `DEBUG_ROUTES=true` | Enables `/debug/*` endpoints |

Set these in `.dev.vars` (git-ignored). Never commit `.dev.vars`. Use `.dev.vars.example` as the template.

## Key Patterns

### Running CLI Commands in the Container

Always include `--url ws://localhost:18789`. CLI is named `clawdbot`:

```typescript
const proc = await sandbox.startProcess(
  'clawdbot devices list --json --url ws://localhost:18789'
);
```

CLI commands take **10–15 seconds** due to WebSocket overhead. Use the `waitForProcess()` helper from `src/gateway/utils.ts`.

### Success Detection

The CLI outputs `"Approved"` (capital A). Use case-insensitive check:

```typescript
stdout.toLowerCase().includes('approved')
```

### Adding a New API Endpoint

1. Add route handler in `src/routes/api.ts`
2. Add or update types in `src/types.ts` if needed
3. Update `src/client/api.ts` if the frontend needs it
4. Add tests in `src/routes/api.test.ts` (or colocated)

### Adding a New Environment Variable

1. Add to `MoltbotEnv` interface in `src/types.ts`
2. If the variable should be passed to the container, add mapping to `buildEnvVars()` in `src/gateway/env.ts`
3. Update `.dev.vars.example`
4. Document in `README.md` secrets table

## Testing

Tests use **Vitest**. Test files are colocated with source files using the `*.test.ts` suffix.

```
src/auth/jwt.test.ts           # JWT decoding and validation
src/auth/jwks.test.ts          # JWKS fetching and caching
src/auth/middleware.test.ts    # Auth middleware behavior
src/gateway/env.test.ts        # Container env var building
src/gateway/process.test.ts    # Process find/start logic
src/gateway/r2.test.ts         # R2 mounting
src/gateway/sync.test.ts       # R2 backup/restore
src/logging.test.ts            # Log redaction of sensitive params
```

**Shared mock utilities** are in `src/test-utils.ts`:
- `createMockEnv()` — mock Worker bindings
- `createMockEnvWithR2()` — mock with R2 credentials
- `createMockProcess()` — mock sandbox process with configurable output
- `createMockSandbox()` — mock sandbox with configurable behavior
- `suppressConsole()` — silence logs during test runs

**E2E tests** live in `test/e2e/` and use Playwright.

Add tests for all new functionality.

## Code Style

- TypeScript strict mode — no `any` unless unavoidable
- Prefer explicit types over inference for function signatures
- Keep route handlers thin — extract business logic to `src/gateway/` modules
- Use Hono's context methods (`c.json()`, `c.html()`, `c.text()`) for responses
- Do not add comments where the logic is self-evident
- Do not add error handling for scenarios that cannot occur

## Container & Docker

**Base image:** `docker.io/cloudflare/sandbox:0.7.0`

Key additions:
- Node.js upgraded from 20 → 22.13.1 (clawdbot requirement)
- Installs: `rsync`, `pnpm`, `clawdbot`
- Exposes port `18789` (Moltbot gateway)
- Startup via `start-moltbot.sh` → configures from env vars → launches gateway with `--allow-unconfigured`

### Docker Cache Busting

When changing `moltbot.json.template` or `start-moltbot.sh`, bump the cache bust comment in `Dockerfile`:

```dockerfile
# Build cache bust: 2026-01-28-v26-browser-skill
```

## Gateway Configuration

Moltbot config is assembled at container startup:

1. `moltbot.json.template` is copied to `~/.clawdbot/clawdbot.json`
2. `start-moltbot.sh` patches config with env var values
3. Gateway starts with `--allow-unconfigured` (skips onboarding wizard)

### Container Environment Variable Mappings

| Worker Secret | Container Env Var | Notes |
|--------------|-------------------|-------|
| `ANTHROPIC_API_KEY` | `ANTHROPIC_API_KEY` | Direct pass-through |
| `MOLTBOT_GATEWAY_TOKEN` | `CLAWDBOT_GATEWAY_TOKEN` | Name change |
| `DEV_MODE` | `CLAWDBOT_DEV_MODE` | Name change |
| `TELEGRAM_BOT_TOKEN` | `TELEGRAM_BOT_TOKEN` | Direct |
| `DISCORD_BOT_TOKEN` | `DISCORD_BOT_TOKEN` | Direct |
| `SLACK_BOT_TOKEN` | `SLACK_BOT_TOKEN` | Direct |
| `SLACK_APP_TOKEN` | `SLACK_APP_TOKEN` | Direct |

### Moltbot Config Schema Gotchas

- `agents.defaults.model` must be `{ "primary": "model/name" }` — not a plain string
- `gateway.mode` must be `"local"` for headless operation
- There is no `webchat` channel — the Control UI is served automatically
- `gateway.bind` is not a config option — use the `--bind` CLI flag instead

## R2 Storage Gotchas

R2 is mounted via `s3fs` at `/data/moltbot`.

- **Use `rsync -r --no-times`**, not `rsync -a`. s3fs does not support setting timestamps; `-a` causes "Input/output error".
- **Check mount status explicitly**: Do not rely on `sandbox.mountBucket()` error messages to detect an already-mounted state. Instead run `mount | grep s3fs`.
- **Never delete `/data/moltbot/*`**: That directory IS the R2 bucket. Deleting it deletes your backup.
- **Process status timing**: `proc.status` may not update immediately after a process completes. Verify success by checking for expected output (e.g., a timestamp file), not by polling `proc.status`.

## Documentation Files

| File | Audience | Contents |
|------|----------|---------|
| `README.md` | End users | Setup, deployment, secrets, chat channels, troubleshooting |
| `AGENTS.md` | AI agents | Legacy guidelines (superseded by this file) |
| `CONTRIBUTING.md` | Contributors | Contribution rules, AI disclosure requirements |
| `CLAUDE.md` | AI assistants | This file |

**Rule:** Development documentation goes here or in `AGENTS.md`, not `README.md`.

## Contributing Rules (from CONTRIBUTING.md)

- Open an issue before starting non-trivial work
- Demonstrate testing in your PR (manual or automated)
- AI usage must be disclosed in PRs
- AI-generated PRs are only accepted for pre-approved issues
- AI contributions must be fully human-verified before merging
- No AI-generated media (images, audio, video)
