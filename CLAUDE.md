# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

# CSheet Project Guidelines

CSheet is an open-source D&D 5e character sheet app (self-hostable). Uses Hono + JSX SSR + htmx for interactivity, Bun as the runtime, and PostgreSQL for persistence.

## Runtime: Bun

Always use Bun instead of Node.js equivalents:

- `bun <file>` not `node`/`ts-node`
- `bun test` not jest/vitest
- `bun install` not npm/yarn/pnpm
- `Bun.sql` for PostgreSQL, `Bun.file` for files, `Bun.$` instead of execa
- Bun auto-loads `.env` — don't use dotenv

## Common Commands

```bash
mise run app:dev          # Start dev server with hot reload (starts deps automatically)
mise run app:container    # Build and run app + all services in Docker
mise run test             # Run tests (sets up test DB automatically)
mise run check            # Biome lint/format + TypeScript type check
mise run check-fix        # Auto-fix lint/format issues
mise run db:upgrade       # Run pending migrations
mise run dbmate new <name> # Create a new migration
mise run deps:up          # Start PostgreSQL + MinIO in Docker
mise run deps:down        # Stop Docker services
```

Run a single test file: `mise run test src/routes/character.test.ts`
Run tests matching a pattern: `mise run test --test-name-pattern "when user is authenticated"`

## Architecture

### Request Flow

```
main.ts → createApp(db?) → Hono app
  ├── Health/webhook routes (no middleware)
  ├── jsxRenderer middleware (wraps all responses in Layout)
  ├── applyMiddleware() — auth, flash, logging, etc.
  ├── Public routes: /, /login, /spells, /beasts
  └── Protected routes (requireAuth): /characters, /campaigns, /chat, /profile, /uploads
```

### Key Patterns

**SSR + htmx**: Pages are server-rendered JSX. Dynamic interactions (form submissions, partial updates) use htmx — the server returns HTML fragments that htmx swaps into place. No client-side state management.

**Database injection**: `createApp(db?)` accepts an optional `Bun.sql` instance. Tests inject a transaction-scoped connection for isolation. Production uses the default connection from `src/db.ts`.

**Context variables**: Set by middleware, accessed in routes via `c.get("user")`, `c.get("flash")`, `c.get("notifications")`, `c.get("db")`.

**Services layer**: Business logic lives in `src/services/` (e.g., `computeCharacter.ts` derives all character stats from raw DB data, `longRest.ts`, `createCharacter.ts`). Routes should call services, not contain logic.

**D&D Rulesets**: `src/lib/dnd/` contains a pluggable ruleset system. Each character has a `ruleset` field selecting SRD 5.1 (2014) or SRD 5.2 (2024). `rulesets.ts` loads the correct ruleset object; `srd51.ts`/`srd52.ts` define classes, spells, species, etc.

### Project Structure

```
src/
├── app.ts              # createApp() — route registration, middleware setup
├── config.ts           # All env vars with defaults (also prints JSON when run directly)
├── db.ts               # Default Bun.sql connection
├── middleware.ts       # Applies all middleware (auth, flash, logging, etc.)
├── components/         # JSX page components + ui/ for reusable pieces
├── routes/             # Route handlers (one file per feature area)
├── middleware/         # auth.ts, flash.ts, cachingServeStatic.ts
├── db/                 # DB models — findById, findByEmail, create, etc.
├── services/           # Business logic (computeCharacter, longRest, etc.)
├── lib/
│   ├── dnd/            # D&D rules engine (rulesets.ts, srd51.ts, srd52.ts)
│   ├── schemas.ts      # Zod validation schemas
│   └── logger.ts       # Structured logger (human-readable dev, JSON in prod)
└── test/               # Test infrastructure
    ├── app.ts          # useTestApp() helper
    ├── http.ts         # makeRequest, parseHtml, expectElement helpers
    └── factories/      # fishery + faker factories for test data
```

## Framework Conventions

**Routes**: Export a named `Hono` instance, register in `app.ts`. Use `c.render(<Component />, { title: "..." })` to render with Layout.

**Components**: Bootstrap 5 for styling, typed props interface, `class=` not `className=`. Accept `user?: User` when auth state needed.

**Layout**: Automatically receives `user`, `currentPage`, `flash`, `notifications` from middleware — routes don't need to pass these.

## Authentication

Signed cookies via `src/middleware/auth.ts`. Sets `c.var.user`. Use `setAuthCookie(c, userId)` / `clearAuthCookie(c)` from the auth module. Auth uses OTP (email-based) — no passwords.

## Database

PostgreSQL 16 in Docker. Migrations managed by dbmate in `migrations/`. Schema auto-generated at `db/schema.sql`.

```env
POSTGRES_HOST=localhost
POSTGRES_PORT=5432
POSTGRES_USER=csheet_user
POSTGRES_PASSWORD=csheet_pass
POSTGRES_DB=csheet_dev
```

## Docker

The compose file has two profiles:

- **Default** (no profile): starts `postgres` and `minio` only — used for local dev
- **`app` profile**: additionally builds/runs the app container with migrations

```bash
# Deps only (local dev)
docker compose up --detach --wait

# Full stack in Docker (equivalent to mise run app:container)
docker compose --profile app up --build app

# Stop full stack
docker compose --profile app down
```

The `migrations` service runs dbmate before `app` starts (`service_completed_successfully` dependency).

## Self-Hosting Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `COOKIE_SECRET` | dev default | Must be set in production |
| `SHOW_WELCOME_PAGE` | `true` | Show onboarding to new users |
| `ANTHROPIC_API_KEY` | — | Enables AI chat features |
| `RESEND_API_KEY` | — | Email delivery via Resend |
| `SMTP_HOST` | — | Alternative: SMTP email delivery |
| `S3_ENDPOINT` | `http://localhost:9000` | MinIO/S3 for file uploads |

## Logging

Use `logger` from `src/lib/logger.ts` instead of `console.*`:

```typescript
logger.info("message", { key: value })
logger.error("message", error as Error, { key: value })
logger.warn("message", { key: value })
```

Dev: human-readable prefixed output. Production: JSON for Google Cloud Logging.

## Testing

Integration tests only — test through HTTP, not individual functions. Each test runs inside a rolled-back PostgreSQL transaction.

```typescript
import { useTestApp } from "@src/test/app"
import { userFactory } from "@src/test/factories/user"
import { makeRequest, parseHtml, expectElement } from "@src/test/http"

describe("GET /characters", () => {
  const testCtx = useTestApp()  // Never destructure — values set in beforeEach

  describe("when user is authenticated", () => {
    let user: User
    beforeEach(async () => {
      user = await userFactory.create({}, testCtx.db)
    })

    test("displays characters", async () => {
      const response = await makeRequest(testCtx.app, "/characters", { user })
      const document = await parseHtml(response)
      expectElement(document, ".character-list")
    })
  })
})
```

Tests use a separate `csheet_test` database. `mise run test` creates/migrates it automatically.

## Code Quality

Biome for linting/formatting (`biome.json`). TypeScript for types (`tsconfig.json`). No `*` imports — import only what you need.

## Deployment

Production: Google Cloud Run via Pulumi (`pulumi/infra/` for VPC/Cloud SQL, `pulumi/app/` for the service). Triggered automatically by GitHub releases via Workload Identity Federation.

```bash
mise run infra:up      # Apply infrastructure
mise run deploy:push   # Build + push image (tagged with commit SHA)
mise run deploy:up     # Deploy to Cloud Run
mise run ops:logs      # Tail production logs
```
