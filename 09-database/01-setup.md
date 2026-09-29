# Module 37 — Postgres + Drizzle: The Data Stack, Set Up

**Phase 9: Database & ORM · Module 37 of 101**

> **Where does this run?** Everything in this module is **`[SERVER]`** — the database connection is the *furthest* server-only thing in the app (it holds the credentials). The `server-only` guard (the "module 09-01" anchor the earlier modules pointed at) is *defined here*: `import 'server-only'` in `src/db/` makes any client-file import of the DB a **build error**.

---

## 1. Concept — The verified stack and the one rule that owns it

**The verified modern stack** (module 01's table, this row — September 2026, official sources):

| Technology | Version | Why (the README's decision matrix, this row) |
|---|---|---|
| **PostgreSQL** | **18** (released 2025-09-25) | The course's primary DB (the spec's default: Postgres unless evidence says otherwise). The extensions (JSONB, `gen_random_uuid()`, `pg_trgm` for search — Phase 17, `pg_cron` for jobs — Phase 23) make it the full-stack default |
| **Drizzle** | **v1** (stable) | The course's primary ORM (the README's decision matrix). *Typed end-to-end* (the schema *is* the TypeScript — the module-04's "Zod vs TS" line, the DB version: the Drizzle schema's *type* flows to the query's result), *SQL-visible* (every query is inspectable SQL — the module-18's "measure where it runs" line: you can `EXPLAIN` what Drizzle sends), *migrations as plain SQL* (the module-38's: auditable, diff-able, no black box). The alternative is Prisma 7 (module 42 — the ⚡ module) |
| **drizzle-kit** | **0.8** | The schema → migration generator + the migration applier (`generate`/`migrate`/`push` — module 38's) |
| **node-postgres (`pg`)** | latest stable | The driver behind Drizzle's `node-postgres` dialect (the connection pool — module 41's subject) |

**The one rule that owns the whole phase** (module 17's five contract rules, rule 1, now with a *mechanism*): **the service layer is the only code that touches the ORM** (`src/services/**` imports `src/db/`; the pages/actions/Route Handlers import the *services*, never `src/db/` — module 17's "collapsed data path"). The *mechanism* that makes the rule a build error instead of a convention is this module's §4's `import 'server-only'` — the **module-09-01 guard** the course has been pointing at:

```
client file imports src/db (directly or transitively)
  → the 'server-only' package throws at BUILD time
  → the build fails
  → the rule is enforced (module 14's wire format's security property, at the DB level)
```

## 2. Mental Model — The data path, four hops (each with a job)

```mermaid
flowchart LR
    S["src/services/*.ts<br/>(module 17 — the logic:<br/>tenancy, DTOs, AppError)"] --> D["src/db/index.ts<br/>(this module — the CLIENT:<br/>one Drizzle instance,<br/>server-only)"]
    D --> P["pg Pool (node-postgres)<br/>(module 41 — the POOL:<br/>connections, not per-query)"]
    P --> PG[("Postgres 18<br/>(module 38 — the SCHEMA:<br/>constraints are the truth)")]
```

| Hop | The job | The failure mode when skipped |
|---|---|---|
| **Service → db client** (module 17) | The tenancy (the `orgId`-first), the DTO mapping (module 17's rule 3), the `AppError` (module 17's rule 4) | The tenancy check scattered per page (module 11-02's cross-tenant bug); the raw rows in the DTO (module 14's leak) |
| **Drizzle** (the typed query builder) | The *type-safe* query (the module-04's line: the schema's type → the result's type) + the *SQL you can read* (module 18's) | The string-SQL (the module-19's injection surface — the *parameterized* Drizzle query is the control) |
| **pg Pool** (module 41) | The *connection* (not per-query — a *pool*; module 41's whole subject) | The per-query connection (module 41's "serverless breaks naive pools" — the connection exhaustion) |
| **Postgres** (the constraints) | The *last line of truth* (the FK, the `CHECK`, the `UNIQUE` — module 38's: the DB constraint is the *final* tenancy/integrity guard, *after* the service's check) | The service's check as the *only* guard (the module-19's line: the *defense in depth* — the DB constraint is the *last* layer; the service's check is the *first*) |

**The mental-model sentence** (the module's thesis): *the service decides (tenancy, DTO), Drizzle asks (typed, parameterized), the pool connects (pooled, not per-query), Postgres enforces (constraints, the last line)* — **four jobs, four layers, no layer does another's job** (module 17's rule 1 is the *service's* job; the DB constraint is *not* the service's job (the *module-38's* line: the constraint is the *last line* (module 38's) — the *service's check is the first* (module 17's) — the *two are separate* (the defense in depth, module 19's))).

## 3. Architecture — Local Postgres with Docker (the dev truth)

### 3.1 The `docker-compose.yml` (the local PG 18, pinned)

`FILE: docker-compose.yml` (production pattern — the dev truth; the *pinned* image (module 02's verified stack: PG **18**))

```yaml
# The LOCAL Postgres (module 37's dev truth) — the PINNED image (module 01's verified: PG 18):
services:
  db:
    image: postgres:18            # the module-01's verified version (the 2026-09-25 release)
    restart: unless-stopped
    environment:
      POSTGRES_USER: capstone          # the module-19's least privilege: the app's user (module 37's §6)
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-capstone-dev}   # the module-02's 4-split env: the DEV default (module 02's)
      POSTGRES_DB: capstone
    ports:
      - '5432:5432'                   # the local port (module 02's: the 5432 — the Postgres default)
    volumes:
      - db-data:/var/lib/postgresql/data   # the persisted data (the dev DB survives the restart)
    healthcheck:
      test: ['CMD-SHELL', 'pg_isready -U capstone -d capstone']
      interval: 5s
      timeout: 3s
      retries: 10
    # the module-41's local dev: the pg pooler is NOT needed locally (module 41's: the local is one process) —
    # the PgBouncer service is the DEPLOYMENT's (module 22's) — the module-37's line: the local is simple (module 41's)

volumes:
  db-data:
```

**The module-37's line:** the *local* Postgres is *one* Docker container (module 37's) — the *pooler* (module 41's) is the *deployment's* concern (module 22's) — the *no local PgBouncer* (module 41's line: the local is one process — the pooler is the serverless's/module 22's).

### 3.2 The env wiring (module 02's 4-split, the DB's slice)

Module 02's env 4-split (the module-02's verified: the *server-only env*): the `DATABASE_URL` is a **server-only secret** (module 02's 4-split: the *server* env — *never* the `NEXT_PUBLIC_` prefix). The module-04's boot-parse (the module-04's verified: the *Zod env schema at startup*):

`FILE: src/env.ts` (production pattern — [SERVER] — the module-04's boot-parse, the DB's slice)

```ts
import { z } from 'zod'

// the module-04's env schema (the BOOT parse — the module-02's 4-split, the SERVER slice):
// the DATABASE_URL is SERVER-ONLY (module 02's 4-split: the no NEXT_PUBLIC_ — module 02's verified):
const serverEnv = z.object({
  NODE_ENV: z.enum(['development', 'test', 'production']),
  DATABASE_URL: z.string().url().startsWith('postgres://', { message: 'DATABASE_URL must be a postgres:// URL' }),
  // the module-41's pool settings (module 41's: the pool's max — the local's default, the deployment's override):
  DB_POOL_MAX: z.coerce.number().min(1).max(100).default(10),
  // the module-19's: the better-auth's secret (Phase 10) — the placeholder here (the Phase 10's wiring):
  // BETTER_AUTH_SECRET: z.string().min(32),
})
// the module-04's line: the parse at BOOT (the module-02's: the fail-fast — the no runtime env error):
export const env = serverEnv.parse(process.env)   // throws at startup if the env is invalid (module 04's)
```

`FILE: .env` (the dev truth — the module-02's 4-split: the server slice; the `.env` is git-ignored — module 02's verified)

```bash
# the DEV env (module 02's 4-split: the SERVER slice — the .env is GIT-IGNORED (module 02's))
NODE_ENV=development
DATABASE_URL=postgres://capstone:capstone-dev@localhost:5432/capstone
DB_POOL_MAX=10
```

**The module-37's line:** the `DATABASE_URL` is the *server-only* secret (module 02's 4-split) — the *boot parse* (module 04's) — the *fail-fast* (module 02's) — the *no runtime env error* (module 04's line).

## 4. Production Code — The Drizzle client (the `server-only` guard, defined)

`FILE: package.json` (the scripts — the module-38's migration workflow, previewed)

```jsonc
{
  "scripts": {
    "dev": "next dev",
    "db:up": "docker compose up -d db",                          // the local PG (module 37's §3.1)
    "db:down": "docker compose down",
    "db:generate": "drizzle-kit generate",                       // the module-38's: the schema → the SQL migration
    "db:migrate": "drizzle-kit migrate",                         // the module-38's: the apply the migration
    "db:push": "drizzle-kit push",                               // the DEV-ONLY (module 38's: the push is the dev's truth — the no push in prod)
    "db:studio": "drizzle-kit studio",                           // the module-38's: the DB's UI (the dev's inspection)
    "db:seed": "tsx src/db/seed.ts"                              // the module-38's: the seed (the dev's data)
  }
}
```

`FILE: drizzle.config.ts` (the drizzle-kit's config — the module-38's migration's home)

```ts
import { defineConfig } from 'drizzle-kit'

// the drizzle-kit's config (module 38's: the migration's generator + applier):
// the dialect: 'postgresql' (module 01's verified: PG 18) — the schema: the module-38's src/db/schema.ts:
export default defineConfig({
  dialect: 'postgresql',
  schema: './src/db/schema.ts',        // the module-38's: the Drizzle schema (the single source of truth)
  out: './drizzle',                     // the migration's SQL (the module-38's: the plain SQL — the auditable)
  dbCredentials: {
    url: process.env.DATABASE_URL!,     // the module-04's env (the boot parse — the module-37's §3.2)
  },
  verbose: true,
  strict: true,
})
```

`FILE: src/db/index.ts` (the Drizzle client — the module-37's core artifact — the `server-only` guard, defined)

```ts
// THE DRIZZLE CLIENT (module 37's core — the ONE instance, the server-only guard):
// the module-09-01 GUARD (the course's standing anchor — the module-14's wire format's security property, at the DB level):
import 'server-only'   // THE GUARD (module 37's §1's line): importing src/db from a CLIENT file = BUILD ERROR.
                       // the 'server-only' package (npm i server-only) throws a build-time error when bundled
                       // into a client chunk — the module-17's rule 1 (the service is the only ORM code) is
                       // ENFORCED (the module-14's wire format's security property, at the DB level):

import { drizzle } from 'drizzle-orm/node-postgres'   // the module-01's verified: the Drizzle v1's node-postgres dialect
import { Pool } from 'pg'                              // the module-01's verified: the node-postgres driver (the pg Pool — module 41's)
import * as schema from './schema'                     // the module-38's: the schema (the typed relations)
import { env } from '@/env'                            // the module-04's boot parse (the module-37's §3.2)

// the module-41's POOL (module 41's: the pg Pool — the connections, NOT per-query):
// the local's max (module 37's §3.2's DB_POOL_MAX) — the deployment's override (module 41's):
const pool = new Pool({
  connectionString: env.DATABASE_URL,
  max: env.DB_POOL_MAX,        // the module-41's: the pool's max (the local's 10 — the deployment's module 41's number)
  // the module-41's: the idle timeout (the serverless's module 22's concern — module 41's deep-dive):
  // idleTimeoutMillis: 30_000,
  // the module-19's: the SSL (the deployment's — the local's no SSL (module 37's line: the local is simple)):
  // ssl: env.NODE_ENV === 'production' ? { rejectUnauthorized: false } : undefined,
})

// THE DRIZZLE INSTANCE (the module-38's: the schema is the TYPE — the query's result is the schema's type):
export const db = drizzle(pool, { schema })   // the module-01's verified: the drizzle v1's signature (the pool + the schema)
export type Db = typeof db                     // the module-04's: the typed db (the service's parameter's type)

// the module-19's line: the pool's CLOSE (the graceful shutdown — the module-22's deploy's concern, previewed):
// (the src/instrumentation.ts's module-19's tracer's shutdown hook — the Phase 21's wiring: pool.end())
```

`FILE: src/db/health.ts` (the smoke test — the module-37's "it works" proof)

```ts
// THE SMOKE TEST (module 37's: the "it works" proof — the service's version, module 17's rule 1):
import 'server-only'   // the module-37's §1's guard (the no client import)
import { db } from './index'
import { sql } from 'drizzle-orm'

export async function checkDbHealth(): Promise<{ ok: boolean; version: string | null }> {
  try {
    const [row] = await db.execute(sql`select version() as v`)   // the module-18's: the SQL is visible (the drizzle's sql template)
    return { ok: true, version: String(row?.v ?? null) }
  } catch {
    return { ok: false, version: null }   // the module-19-03's: the no error detail (the module-21's digest's separate log)
  }
}
```

**Wiring the health into the module-34's `GET /api/health`** (the module-34's §8's beginner exercise, the *real* version):

```ts
// the module-34's health endpoint (module 34's §8's beginner, the REAL version):
import { NextResponse } from 'next/server'
import { checkDbHealth } from '@/db/health'   // the module-37's smoke test (the module-17's line: the health is a SERVICE (module 17's rule 1) — the no db import in the route)

export async function GET() {
  const { ok, version } = await checkDbHealth()   // the module-37's §4's smoke test
  return NextResponse.json(
    { status: ok ? 'ok' : 'degraded', db: ok ? version : null },
    { status: ok ? 200 : 503 },   // the module-36's: the 503 (the degraded — the module-34's §3.2's table's 500's *degraded* version)
  )
}
```

### 4.1 The `server-only` guard, proven (the module-09-01 anchor, exercised)

The *proof* (the module-37's line: the guard is the *build error* — the *no convention* (module 17's rule 1, enforced)):

```bash
# the module-37's §4.1's PROOF (the build error — the module-14's wire format's security property, at the DB level):
# (a) the guard WORKS: a client file imports src/db → the BUILD fails:
echo "import { db } from '@/db'" > src/components/evil.tsx   # the [CLIENT] file (the no 'use client'? the import alone is enough — the 'server-only' throws at BUNDLE time)
npm run build
# → the build error (the module-37's §1's line): "This module cannot be imported from a client component" (the 'server-only' package's build-time throw)
# (b) the guard is the STANDARD: the service imports src/db → the build PASSES (the [SERVER] file):
# (the src/services/*.ts's import { db } from '@/db' — the module-17's rule 1 — the build passes)
# (c) the guard is the WIRE's security: the module-14's line (the wire format's security property) is the DB's version (module 37's §1)
```

**The module-37's standing line:** the `server-only` guard is the *module-14's wire format's security property, at the DB level* (module 37's §1) — the *build error* (the no convention) (module 17's rule 1, enforced) — the *the guard is proven* (module 37's §4.1: the build error + the build pass) — the *module-09-01's anchor, defined* (module 37's).

## 5. Common Mistakes (the setup failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The per-query connection** (the `new Client()` per query — the module-41's line violated) | The *connection exhaustion* (module 41's: the serverless's — the *module-41's whole subject*) — the *module-37's line: the pool is the module-41's* (module 41's) — the *no per-query* (module 41's) | The *pg Pool* (module 37's §4) — the *module-41's* *deep-dive* (module 41's) — the *no per-query connection* (module 41's line) |
| **The `DATABASE_URL` in the `NEXT_PUBLIC_`** (module 02's 4-split violated) | The *leak* (module 19-03's: the `NEXT_PUBLIC_` is the *client bundle* (module 02's verified) — the *DB's URL is the *secret* (module 19's) — the *module-37's line: the `DATABASE_URL` is the server-only* (module 02's 4-split) — the *no `NEXT_PUBLIC_`* (module 02's)) | The *server env* (module 02's 4-split) — the *no `NEXT_PUBLIC_`* (module 02's) — the *module-04's boot parse* (module 37's §3.2) |
| **The runtime env read** (the `process.env.DATABASE_URL` in the *service* — the module-04's boot-parse violated) | The *runtime env error* (module 04's: the *fail-fast* violated — the *module-02's* *line: the boot parse is the fail-fast* (module 02's) — the *module-37's line: the env is the boot parse* (module 04's) — the *no runtime* (module 04's)) | The *module-04's boot parse* (module 37's §3.2) — the *fail-fast* (module 02's) — the *no runtime env error* (module 04's line) |
| **The db import in the route** (the module-17's rule 1 violated: the route's `import { db } from '@/db'` + the query) | The *module-34's 4-things-it-is-not* (module 34's §1: the *DB in the route* — the *module-17's rule 1* violated) — the *tenancy* *missing* (module 11's) — the *module-37's line: the service is the only ORM code* (module 17's rule 1) — the *no db in the route* (module 17's rule 1) | The *module-17's service* (module 17's rule 1) — the *module-34's 4-things* (module 34's §1) — the *no db in the route* (module 17's rule 1) |
| **The unpinned PG image** (the `postgres:latest` — the module-01's verified stack violated) | The *module-01's line: the verified version* (module 01's) — the *`latest` is the *unverified* (module 01's) — the *module-37's line: the image is the *pinned* (module 01's verified: PG 18) — the *no `latest`* (module 01's) | The *`postgres:18`* (module 37's §3.1) — the *module-01's verified* (module 01's) — the *no `latest`* (module 01's line) |
| **The `drizzle-kit push` in prod** (module 38's line violated: the push is the dev's truth) | The *module-38's line: the push is the dev's truth* (module 38's) — the *the prod is the *migrate* (module 38's) — the *no push in prod* (module 38's) — the *module-38's §4's line: the migration is the *history* (module 38's) — the *push is the *no history* (module 38's))* | The *`drizzle-kit migrate`* (module 38's) — the *the prod is the *migrate* (module 38's) — the *no push in prod* (module 38's line)* |
| **The db connection in the client bundle** (the `import { db }` in the `'use client'` — the module-37's §4.1's guard violated) | The *build error* (module 37's §4.1) — the *the guard is the *build error* (module 37's §1) — the *no client db* (module 37's §1)* | The *module-37's §4.1's guard* (module 37's §1) — the *build error* (module 37's §1) — the *no client db* (module 37's §1) |

## 6. Security Notes

- **The `DATABASE_URL` is the server-only secret** (module 02's 4-split): the *no `NEXT_PUBLIC_`* (module 02's) — the *the URL is the *credential* (module 19's) — the *module-37's line: the `DATABASE_URL` is the server-only* (module 02's 4-split) — the *no client* (module 02's)*.
- **The least-privilege DB user** (module 19's line: the *app's user* (module 37's §3.1: the `capstone` user) — the *no `postgres` superuser for the app* (module 19's) — the *the migrations run as the *migration user* (module 38's: the *separate* user (module 19's line: the *migration's privilege* (module 19's) — the *app's user is the *least* (module 19's))* — the *module-37's line: the app's user is the least-privilege* (module 19's) — the *migration's user is the separate* (module 19's)*.
- **The `server-only` guard** (module 37's §1): the *the client's db is the *build error* (module 37's §1) — the *module-14's wire format's security property, at the DB level* (module 37's §1) — the *the guard is the *enforcement* (module 17's rule 1)*.
- **The env's secret at rest** (module 19's line: the *the `.env` is the git-ignored* (module 02's) — the *the prod's secret is the *platform's* (module 22's: the Vercel's env / the Docker's secret (module 22's) — the *module-37's line: the prod's secret is the platform's* (module 22's) — the *no `.env` in prod* (module 22's)*.

## 7. Performance Notes

- **The pool is the module-41's** (module 37's §4: the pg Pool — the module-41's deep-dive): the *local's max* (module 37's §3.2's `DB_POOL_MAX=10`) — the *deployment's* (module 41's: the *serverless's* *pool* (module 22's) — the *module-41's* *line: the local is the one process* (module 37's) — the *deployment is the pooler* (module 41's))*.
- **The connection is not per-query** (module 41's line): the *the pool's connection is the *reused* (module 41's) — the *the per-query is the *module-41's failure* (module 41's) — the *module-37's line: the pool is the module-41's* (module 41's) — the *no per-query* (module 41's)*.
- **The query's cost is the module-18's** (module 39's: the *the query's* *cost* (module 18's) — the *module-39's* *line: the query is the module-18's* (module 18's) — the *module-37's line: the query's cost is the module-18's* (module 18's) — the *module-39's* *deep-dive* (module 39's))* — the *the `EXPLAIN` is the module-18's* (module 18's) — the *module-39's* *line: the `EXPLAIN` is the module-18's* (module 18's)*.

## 8. Exercise

**Beginner.** *The local stack* (module 37's §3–4): the *`docker-compose.yml`* (module 37's §3.1) + the *`.env`* (module 37's §3.2) + the *`src/env.ts`* (module 37's §3.2) + the *`drizzle.config.ts`* (module 37's §4) + the *`src/db/index.ts`* (module 37's §4) + the *`src/db/health.ts`* (module 37's §4) — *build it* — the *`npm run db:up`* — the *`npm run db:push`* (the module-38's: the empty schema's push — the *the "it works"* (module 37's)) — the *health endpoint* (module 34's §8's beginner, the real version) — the *curl* (the *200* + the *version*) — the *artifact: the curl's output* (module 20's).

**Intermediate.** *The `server-only` guard's proof* (module 37's §4.1): the *`src/components/evil.tsx`* (the `import { db } from '@/db'`) — the *`npm run build`* — the *build error* (module 37's §4.1) — the *delete the evil.tsx* — the *build passes* — the *artifact: the two build outputs (the error + the pass)* (module 20's) — the *module-37's §4.1's line: the guard is the build error* (module 37's §1).

**Production.** *The health's full test* (module 37's §4's health, the *module-34's* §8's production): the *PG up* (the *200* + the version) — the *PG down* (the *503* + the `null`) — the *the `DATABASE_URL` wrong* (the *boot parse's fail-fast* (module 04's) — the *the app doesn't start* (module 02's fail-fast)) — the *artifact: the three outputs (the 200, the 503, the boot error)* (module 20's).

## 9. Architecture Challenge

**Prompt:** The *"the team wants to use a 'database client' in the frontend for 'flexibility'"* (the *the client's db* (module 37's §5's mistake #7) — the *module-37's line: the client's db is the build error* (module 37's §1) — the *the team's "flexibility" is the *module-14's wire format's violation* (module 14's) — the *module-37's line: the client's db is the build error* (module 37's §1) — the *the team's "flexibility" is the module-14's violation* (module 14's)*.

The *problems*: (1) the *the client's db is the build error* (module 37's §1) — the *the `server-only` guard is the *module-14's wire format's security property* (module 37's §1) — the *module-37's line: the client's db is the build error* (module 37's §1) — the *the team's "flexibility" is the module-14's violation* (module 14's)*.

(2) the *the "flexibility" is the *module-17's rule 1's violation* (module 17's rule 1: the service is the only ORM code) — the *the client's db is the *service's* *bypass* (module 17's rule 1) — the *module-37's line: the "flexibility" is the module-17's rule 1's violation* (module 17's) — the *the service is the only ORM code* (module 17's rule 1)*.

(3) the *the "flexibility" is the *module-19's line: the *no client secret* (module 19's) — the *the `DATABASE_URL` is the *server-only* (module 02's 4-split) — the *module-37's line: the "flexibility" is the module-19's no-client-secret violation* (module 19's) — the *the `DATABASE_URL` is the server-only* (module 02's 4-split)*.

**Design**: the *the team's "flexibility"* (the *module-37's line: the client's db is the build error* (module 37's §1) — the *the "flexibility" is the module-14's violation* (module 14's) — the *the module-17's rule 1's violation* (module 17's) — the *the module-19's no-client-secret violation* (module 19's) — the *module-37's standing line: the client's db is the build error* (module 37's §1) — the *the "flexibility" is the module-14's/module-17's/module-19's violation* (module 14's/module 17's/module 19's) — the *no client db* (module 37's §1)*.

Produce: the *the team's "flexibility"* (the *module-37's §1's guard* (module 37's) + the *module-17's rule 1* (module 17's) + the *module-19's no-client-secret* (module 19's) — the *module-37's line: the client's db is the build error* (module 37's §1) — the *the "flexibility" is the violation* (module 14's/module 17's/module 19's)) — and the *module-37's standing line: the client's db is the build error* (module 37's §1) — the *the "flexibility" is the violation* (module 14's/module 17's/module 19's) — the *no client db* (module 37's §1).

<details>
<summary>Model answer</summary>
**The team's "flexibility"** (the *module-37's §1's guard* + the *module-17's rule 1* + the *module-19's no-client-secret*):
1. **The client's db is the build error** (module 37's §1): the *the `server-only` guard is the *module-14's wire format's security property* (module 37's §1) — the *the team's "flexibility" is the module-14's violation* (module 14's) — the *module-37's line: the client's db is the build error* (module 37's §1).
2. **The "flexibility" is the module-17's rule 1's violation** (module 17's rule 1): the *the service is the only ORM code* (module 17's rule 1) — the *the client's db is the *service's* *bypass* (module 17's rule 1) — the *module-37's line: the "flexibility" is the module-17's rule 1's violation* (module 17's).
3. **The "flexibility" is the module-19's no-client-secret violation** (module 19's): the *the `DATABASE_URL` is the *server-only* (module 02's 4-split) — the *module-37's line: the "flexibility" is the module-19's no-client-secret violation* (module 19's).
**The generalization** (the *the team's "flexibility"* pattern, the *module's* standing rule): **the *client's db* is the *build error* (module 37's §1) — the *"flexibility" is the module-14's/module-17's/module-19's violation* (module 14's/module 17's/module 19's) — the *no client db* (module 37's §1) — the *module-37's standing line: the client's db is the build error* (module 37's §1) — the *the "flexibility" is the violation* (module 14's/module 17's/module 19's)*.
</details>

## 10. Official Documentation

- Drizzle (the ORM — module 01's verified v1): https://orm.drizzle.team/
- Drizzle Kit (the migration — module 01's verified 0.8): https://orm.drizzle.team/docs/dialects/postgresql
- `server-only` (the guard — module 37's §1): https://www.npmjs.com/package/server-only
- node-postgres (`pg`, the Pool — module 41's): https://node-postgres.com/
- PostgreSQL 18 (module 01's verified): https://www.postgresql.org/docs/18/
- Drizzle: Getting Started (Postgres): https://orm.drizzle.team/docs/getting-started/postgres
- Drizzle: Migrations: https://orm.drizzle.team/docs/migrations/overview

## 11. What You Should Know Before Continuing

- [ ] I can state the *verified stack* (module 1's table: PG 18, Drizzle v1, drizzle-kit 0.8, `pg`) — the *module-01's verified* (module 01's)
- [ ] I know the *4-hop data path* (module 2's: the service → Drizzle → the pg Pool → Postgres) — the *4 jobs, 4 layers, no layer does another's job* (module 2's line)
- [ ] I can set up the *local stack* (module 3's: the `docker-compose.yml` + the `.env` + the `src/env.ts`) — the *module-02's 4-split* (module 02's) + the *module-04's boot parse* (module 04's)
- [ ] I know the *Drizzle client* (module 4's: the `src/db/index.ts` — the *one instance* — the *the `server-only` guard* (module 37's §1)) — the *module-09-01's anchor, defined* (module 37's)
- [ ] I can prove the *`server-only` guard* (module 4.1's: the *build error* + the *build pass*) — the *module-14's wire format's security property, at the DB level* (module 37's §1)
- [ ] I know the *least-privilege DB user* (module 6's: the app's user + the migration's user) — the *module-19's line* (module 19's)
- [ ] I've done the *local stack* (module 8's beginner) + the *guard's proof* (module 8's intermediate) + the *health's full test* (module 8's production) — the *artifacts: the outputs* (module 20's)
- [ ] I know the *module-41's pool* is the *next module* (module 7's line: the pool is the module-41's) — the *local is the one process* (module 37's) — the *deployment is the pooler* (module 41's)

**Next:** Module 38 — Schema Design & Migrations (the *capstone's data model* — the *multi-tenancy as schema* (the `org_id` + the composite unique) — the *constraints* (the FK, the `CHECK`, the `UNIQUE`) — the *Drizzle schema* (the `pgTable`, the `pgEnum`) — the *migrations* (the `drizzle-kit generate`/`migrate`/`push`) — the *seed*).
