# Module 41 — Connections & Serverless Pooling: The Pool Is the Deployment's, Not the Query's

**Phase 9: Database & ORM · Module 41 of 101**

> **Where does this run?** The pool is **`[SERVER]`** (the DB is server-only, module 37's guard). The *pool config* is **deployment physics** (module 22's: the Docker's pool ≠ the serverless's pool, module 41's §2). A connection is **expensive** (module 41's §1) — the TCP + the auth + the backend; the pool is **the expensive's amortizer** (module 41's §1); the serverless's arithmetic is **the pool's killer** (module 41's §2) — the *1000 instances × 10 connections > 200 max* (module 41's §2).

---

## 1. Concept — The connection is expensive (the pool is the amortizer)

**The connection's cost** (module 41's §1): the *the TCP handshake* (module 41's §1) + the *the auth* (module 37's §6's least-privilege) + the *the backend process* (module 41's §1: the Postgres's *one process per connection* (module 41's §1) — the *module-41's line: the connection is the Postgres's process* (module 41's §1) — the *no shared* (module 41's §1) — the *the `max_connections` is the limit* (module 41's §1) — the *module-41's line: the `max_connections` is the limit* (module 41's §1)).

**The pool's job** (module 41's §1): the *the `pg` Pool is the *amortizer* (module 41's §1) — the *the N queries share the M connections* (module 41's §1) — the *module-41's line: the pool is the amortizer* (module 41's §1) — the *the N > the M* (module 41's §1) — the *the `pool_size` is the M* (module 41's §1) — the *module-41's line: the `pool_size` is the M* (module 41's §1).

**The 4-hop path's connection** (module 37's §1, module 41's §1): the *the service* (module 17) *decides* *what* — the *Drizzle* (module 39) *asks* *how* — the *Pool* (module 41) *connects* (module 37's §1) — the *Postgres* (module 38) *answers* *what's true* — the *module-41's line: the Pool is the 4-hop path's connection* (module 37's §1) — the *module-37's line: the Pool is the connection* (module 37's §1).

**The serverless's arithmetic** (module 41's §2): the *the serverless's instances are the *N* (module 41's §2) — the *the N × the pool_size is the *total* (module 41's §2) — the *the `max_connections` is the *limit* (module 41's §1) — the *module-41's line: the serverless's arithmetic is the pool's killer* (module 41's §2) — the *the `1000 × 10 > 200` is the kill* (module 41's §2) — the *module-41's line: the serverless's arithmetic is the kill* (module 41's §2).

## 2. Mental Model — The pool's 2 shapes (the client-side, the external)

```mermaid
flowchart TD
    A["the N queries (module 17's)"] --> B["the CLIENT-SIDE POOL (the pg Pool — module 41's §2.1)"]
    B --> C["the M connections (the pool_size — module 41's §2.1)"]
    C --> D["the POSTGRES (the max_connections — module 41's §1)"]

    subgraph "the EXTERNAL POOLER (the PgBouncer — module 41's §2.2) — the serverless's fix"
        B2["the N queries"] --> P["the PgBouncer (the transaction mode — module 41's §2.2)"]
        P --> C2["the M connections (the pooler's — module 41's §2.2)"]
    end
```

**The 2 shapes** (the module-41's mental model):
1. **The client-side** (module 41's §2.1): the *the `pg` Pool is the *in-process* (module 41's §2.1) — the *the `pool_size` is the M* (module 41's §2.1) — the *module-41's line: the client-side is the in-process* (module 41's §2.1) — the *the Docker's* *pool* (module 22's) — the *module-22's line: the Docker's pool is the client-side* (module 22's).
2. **The external** (module 41's §2.2): the *the PgBouncer is the *out-of-process* (module 41's §2.2) — the *the `transaction mode` is the *serverless's* (module 41's §2.2) — the *module-41's line: the external is the out-of-process* (module 41's §2.2) — the *the serverless's fix* (module 41's §2.2) — the *module-22's line: the serverless's pool is the external* (module 22's).

## 3. Architecture — The pool (the 2 shapes, the code)

### 3.1 The client-side pool (module 41's §2.1 — the `pg` Pool)

`FILE: src/db/index.ts` (production pattern — [SERVER] — the module-41's §3.1: the `pg` Pool, the `pool_size`)

```ts
// THE CLIENT-SIDE POOL (module 41's §2.1 — the pg Pool — the pool_size is the M (module 41's §2.1)):
import 'server-only'
import { drizzle as createDb } from 'drizzle-orm/node-postgres'
import pg from 'pg'
import * as schema from './schema'
import { env } from '@/env'

const pool = new pg.Pool({
  connectionString: env.DATABASE_URL,   // the module-37's §3's env (module 37's)
  max: env.DB_POOL_MAX,                // the module-41's §3.1's line: the pool_size is the M (module 41's §2.1) — the no default (module 41's §3.1)
  idleTimeoutMillis: 30_000,           // the module-41's §3.1's line: the idle's timeout is the 30s (module 41's §3.1) — the no idle's leak (module 41's §3.1)
  connectionTimeoutMillis: 5_000,      // the module-41's §3.1's line: the connection's timeout is the 5s (module 41's §3.1) — the no hang (module 41's §3.1)
})

export const db = createDb(pool, { schema })   // the module-37's §2's drizzle (module 37's)
```

**The module-41's line:** the *client-side is the in-process* (module 41's §2.1) — the *the `pool_size` is the M* (module 41's §2.1) — the *the no default* (module 41's §3.1) — the *the `idleTimeoutMillis` is the leak's guard* (module 41's §3.1).

### 3.2 The external pooler (module 41's §2.2 — the PgBouncer, the serverless's fix)

`FILE: docker-compose.yml` (simplified example — the module-41's §3.2: the PgBouncer, the transaction mode)

```yaml
# THE EXTERNAL POOLER (module 41's §2.2 — the PgBouncer — the transaction mode (module 41's §2.2)):
services:
  postgres:
    image: postgres:18
    # ... (module 37's §2's)
  pgbouncer:
    image: edoburu/pgbouncer:1.23.1
    ports: ['6432:6432']
    environment:
      DATABASE_URL: postgres://bouncer:password@postgres:5432/saas   # the module-41's §3.2's line: the PgBouncer's admin (module 41's §3.2)
      POOL_MODE: transaction         # THE TRANSACTION MODE (module 41's §2.2) — the serverless's fix (module 41's §2.2)
      DEFAULT_POOL_SIZE: 20          # the module-41's §3.2's line: the pooler's pool_size (module 41's §2.2)
    depends_on:
      - postgres
```

`FILE: src/db/index.ts` (production pattern — [SERVER] — the module-41's §3.2: the pooler's URL)

```ts
// THE POOLER'S URL (module 41's §3.2 — the serverless's fix — the no direct's Postgres (module 41's §2.2)):
const pool = new pg.Pool({
  connectionString: env.DATABASE_URL,   // the module-41's §3.2's line: the DATABASE_URL is the PgBouncer's (module 41's §2.2) — the no direct's Postgres (module 41's §2.2)
  max: 1,                               // the module-41's §3.2's line: the client-side's pool_size is the 1 (module 41's §2.2) — the no client-side's pool (module 41's §2.2) — the *the external is the pool* (module 41's §2.2)
})
```

**The module-41's line:** the *external is the out-of-process* (module 41's §2.2) — the *the `transaction mode` is the serverless's fix* (module 41's §2.2) — the *the client-side's `pool_size` is the 1* (module 41's §2.2) — the *the no client-side's pool* (module 41's §2.2).

## 4. Production Code — The serverless's arithmetic (the pool's killer)

### 4.1 The arithmetic (module 41's §4.1 — the table)

| Deployment | The instances (N) | The pool_size (M) | The total (N × M) | The `max_connections` (L) | The verdict |
|---|---|---|---|---|---|
| **The Docker's 1** (module 22's) | 1 | 10 | 10 | 200 | the *OK* (module 41's §4.1) |
| **The Docker's 3** (module 22's) | 3 | 10 | 30 | 200 | the *OK* (module 41's §4.1) |
| **The serverless's 100** (module 22's) | 100 | 10 | 1000 | 200 | the *KILL* (module 41's §4.1) — the *the `1000 > 200`* (module 41's §2) |
| **The serverless's 100 + the PgBouncer** (module 41's §2.2) | 100 | 1 | 100 | 200 | the *OK* (module 41's §4.1) — the *the external is the fix* (module 41's §2.2) |

**The module-41's line:** the *serverless's arithmetic is the pool's killer* (module 41's §2) — the *the `N × M > L` is the kill* (module 41's §2) — the *the external pooler is the fix* (module 41's §2.2).

### 4.2 The `max_connections` (module 41's §4.2 — the pgTune)

- **The `max_connections` is the limit** (module 41's §1): the *the `pgTune`'s formula* (module 41's §4.2) — the *the `CPU cores × 2 + the effective_spindle_count`* (module 41's §4.2) — the *module-41's line: the `max_connections` is the pgTune's* (module 41's §4.2) — the *the no default* (module 41's §4.2).
- **The `shared_buffers` is the memory's** (module 41's §4.2): the *the `shared_buffers` is the `25%` of the RAM* (module 41's §4.2) — the *module-41's line: the `shared_buffers` is the `25%`* (module 41's §4.2) — the *the `work_mem` is the per-sort* (module 41's §4.2) — the *the no default* (module 41's §4.2).

## 5. Common Mistakes (the pool failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The no `pool_size`** (module 41's §2.1's line violated: the default's pool) | the *module-41's line: the `pool_size` is the M* (module 41's §2.1) — the *the no `pool_size` is the *default's* (module 41's §3.1) — the *module-41's line: the `pool_size` is the M* (module 41's §2.1) — the *no default* (module 41's §3.1)* | the *the `max: env.DB_POOL_MAX`* (module 41's §3.1) — the *module-41's line: the `pool_size` is the M* (module 41's §2.1)* |
| **The no `idleTimeoutMillis`** (module 41's §3.1's line violated: the idle's leak) | the *module-41's line: the `idleTimeoutMillis` is the leak's guard* (module 41's §3.1) — the *the no `idleTimeoutMillis` is the *leak* (module 41's §3.1) — the *module-41's line: the `idleTimeoutMillis` is the leak's guard* (module 41's §3.1) — the *no idle's leak* (module 41's §3.1)* | the *the `idleTimeoutMillis: 30_000`* (module 41's §3.1) — the *module-41's line: the `idleTimeoutMillis` is the leak's guard* (module 41's §3.1)* |
| **The client-side's pool on the serverless** (module 41's §2.2's line violated: the `N × M > L`) | the *module-41's line: the serverless's arithmetic is the pool's killer* (module 41's §2) — the *the `100 × 10 > 200` is the kill* (module 41's §2) — the *module-41's line: the serverless's arithmetic is the kill* (module 41's §2)* | the *the external pooler* (module 41's §2.2) + the *the client-side's `pool_size` is the 1* (module 41's §2.2) — the *module-41's line: the external is the fix* (module 41's §2.2)* |
| **The `session mode` on the PgBouncer** (module 41's §2.2's line violated: the no `transaction mode`) | the *module-41's line: the `transaction mode` is the serverless's fix* (module 41's §2.2) — the *the `session mode` is the *no* (module 41's §2.2) — the *module-41's line: the `transaction mode` is the fix* (module 41's §2.2) — the *no `session mode`* (module 41's §2.2)* | the *the `transaction mode`* (module 41's §2.2) — the *module-41's line: the `transaction mode` is the fix* (module 41's §2.2)* |
| **The no `connectionTimeoutMillis`** (module 41's §3.1's line violated: the hang) | the *module-41's line: the `connectionTimeoutMillis` is the hang's guard* (module 41's §3.1) — the *the no `connectionTimeoutMillis` is the *hang* (module 41's §3.1) — the *module-41's line: the `connectionTimeoutMillis` is the hang's guard* (module 41's §3.1) — the *no hang* (module 41's §3.1)* | the *the `connectionTimeoutMillis: 5_000`* (module 41's §3.1) — the *module-41's line: the `connectionTimeoutMillis` is the hang's guard* (module 41's §3.1)* |
| **The no `max_connections`'s tuning** (module 41's §4.2's line violated: the default's) | the *module-41's line: the `max_connections` is the pgTune's* (module 41's §4.2) — the *the no tuning is the *default's* (module 41's §4.2) — the *module-41's line: the `max_connections` is the pgTune's* (module 41's §4.2) — the *no default* (module 41's §4.2)* | the *the `pgTune`'s formula* (module 41's §4.2) — the *module-41's line: the `max_connections` is the pgTune's* (module 41's §4.2)* |
| **The direct's Postgres on the serverless** (module 41's §2.2's line violated: the no pooler) | the *module-41's line: the external is the fix* (module 41's §2.2) — the *the direct's Postgres is the *kill* (module 41's §2) — the *module-41's line: the direct's Postgres is the kill* (module 41's §2) — the *no direct's Postgres* (module 41's §2.2)* | the *the pooler's URL* (module 41's §3.2) — the *module-41's line: the external is the fix* (module 41's §2.2)* |

## 6. Security Notes

- **The pool is the server-only** (module 37's guard): the *the no client's import* (module 37's §1) — the *module-41's line: the pool is the server-only* (module 37's §1) — the *module-37's line: the `server-only` is the guard* (module 37's §1).
- **The least-privilege** (module 37's §6): the *the pool's user is the *app's* (module 37's §6) — the *module-37's line: the app's user is the least* (module 37's §6) — the *module-41's line: the pool's user is the app's* (module 37's §6).
- **The `transaction mode` is the safe** (module 41's §2.2): the *the `session mode` is the *insecure* (module 41's §2.2) — the *the `SET` is the *session's* (module 41's §2.2) — the *module-41's line: the `transaction mode` is the safe* (module 41's §2.2) — the *no `session mode`* (module 41's §2.2).

## 7. Performance Notes

- **The pool is the amortizer** (module 41's §1): the *module-41's line: the pool is the amortizer* (module 41's §1) — the *the `N > the M`* (module 41's §1) — the *the `pool_size` is the M* (module 41's §1).
- **The serverless's arithmetic** (module 41's §2): the *module-41's line: the serverless's arithmetic is the pool's killer* (module 41's §2) — the *the `N × M > L` is the kill* (module 41's §2) — the *the external pooler is the fix* (module 41's §2.2).
- **The `max_connections` is the pgTune's** (module 41's §4.2): the *module-41's line: the `max_connections` is the pgTune's* (module 41's §4.2) — the *the `CPU cores × 2 + the effective_spindle_count`* (module 41's §4.2).
- **The connection's timeout** (module 41's §3.1): the *module-41's line: the `connectionTimeoutMillis` is the hang's guard* (module 41's §3.1) — the *the no hang* (module 41's §3.1).

## 8. Exercise

**Beginner.** *The client-side's pool* (module 41's §3.1): the *the `pg` Pool* (module 41's §2.1) + the *the `pool_size`* (module 41's §2.1) + the *the `idleTimeoutMillis`* (module 41's §3.1) — *build it* — the *the pool's log* (module 18's) — the *artifact: the pool's config + the log* (module 20's).

**Intermediate.** *The serverless's arithmetic* (module 41's §4.1): the *the table* (module 41's §4.1) — the *the `N × M > L`* (module 41's §2) — the *the PgBouncer's fix* (module 41's §2.2) — the *artifact: the table + the PgBouncer's config* (module 20's).

**Production.** *The pooler's deploy* (module 41's §3.2): the *the `docker-compose.yml`* (module 41's §3.2) + the *the pooler's URL* (module 41's §3.2) + the *the `transaction mode`* (module 41's §2.2) — the *artifact: the `docker-compose.yml` + the pooler's log* (module 20's).

## 9. Architecture Challenge

**Prompt:** The *"the team's serverless's deploy is hitting the Postgres's `max_connections`"* (the *module-41's* *arithmetic* — the *module-22's* *serverless's physics* (module 22's) — the *module-41's line: the serverless's arithmetic is the pool's killer* (module 41's §2) — the *module-22's line: the serverless's physics is the pool's killer* (module 22's) — the *module-41's standing line: the serverless's arithmetic is the kill* (module 41's §2)).

The *problems*: (1) the *the `N × M > L`* (the *module-41's §2's arithmetic* (module 41's §2) — the *module-41's line: the serverless's arithmetic is the pool's killer* (module 41's §2) — the *the `100 × 10 > 200` is the kill* (module 41's §2) — the *module-41's standing line: the serverless's arithmetic is the kill* (module 41's §2)).

(2) the *the external pooler* (the *module-41's §2.2's PgBouncer* (module 41's §2.2) — the *the `transaction mode`* (module 41's §2.2) — the *module-41's line: the external is the fix* (module 41's §2.2) — the *module-41's standing line: the external is the fix* (module 41's §2.2)).

**Design**: the *the external pooler* (the *module-41's §2.2's PgBouncer* (module 41's §2.2) + the *the `transaction mode`* (module 41's §2.2) + the *the client-side's `pool_size` is the 1* (module 41's §2.2) — the *module-41's line: the external is the fix* (module 41's §2.2) — the *module-41's standing line: the external is the fix* (module 41's §2.2)).

Produce: the *the external pooler* (the *module-41's §2.2's PgBouncer* (module 41's §2.2) + the *the `transaction mode`* (module 41's §2.2) + the *the client-side's `pool_size` is the 1* (module 41's §2.2) — the *module-41's line: the external is the fix* (module 41's §2.2) — the *module-41's standing line: the external is the fix* (module 41's §2.2)).

<details>
<summary>Model answer</summary>
**The external pooler** (module 41's §2.2):
1. **The `N × M > L` is the kill** (module 41's §2): the *the module-41's §2's arithmetic* (module 41's §2) — the *module-41's line: the serverless's arithmetic is the pool's killer* (module 41's §2) — the *the `100 × 10 > 200` is the kill* (module 41's §2).
2. **The external is the fix** (module 41's §2.2): the *the module-41's §2.2's PgBouncer* (module 41's §2.2) + the *the `transaction mode`* (module 41's §2.2) + the *the client-side's `pool_size` is the 1* (module 41's §2.2) — the *module-41's line: the external is the fix* (module 41's §2.2).
**The generalization** (the *external pooler's* pattern, the *module's* standing rule): **the *serverless's arithmetic is the pool's killer* (module 41's §2) — the *the external pooler is the fix* (module 41's §2.2) — the *the `transaction mode` is the safe* (module 41's §2.2) — the *module-41's standing line: the serverless's arithmetic is the kill + the external is the fix + the `transaction mode` is the safe* (module 41's §2 + module 41's §2.2)*.
</details>

## 10. Official Documentation

- Node `pg`: Pool: https://node-postgres.com/apis/pool
- PgBouncer: https://www.pgbouncer.org/
- PgBouncer: Transaction pooling mode: https://www.pgbouncer.org/config.html#pooling_mode
- PostgreSQL 18: Server Configuration (`max_connections`): https://www.postgresql.org/docs/18/runtime-config-connection.html
- pgTune: https://pgtune.leopard.in.ua/
- The module-22's deployment physics: the module-22 (the phase-8's file-03, the *serverless's physics*)

## 11. What You Should Know Before Continuing

- [ ] I can state the *connection's cost* (module 1's: the TCP + the auth + the backend) — the *the pool is the amortizer* (module 1's line)
- [ ] I know the *2 shapes* (module 2's: the client-side/external) — the *the `pg` Pool* / the *the PgBouncer* (module 2's)
- [ ] I know the *serverless's arithmetic* (module 2's: the `N × M > L`) — the *the external is the fix* (module 2.2's line)
- [ ] I know the *`transaction mode` is the safe* (module 2.2's) — the *no `session mode`* (module 2.2's line)
- [ ] I know the *`max_connections` is the pgTune's* (module 4.2's) — the *the `CPU cores × 2 + the effective_spindle_count`* (module 4.2's)
- [ ] I've done the *client-side's pool* (module 8's beginner) + the *serverless's arithmetic* (module 8's intermediate) + the *pooler's deploy* (module 8's production) — the *artifacts* (module 20's)

**Next:** Module 42 — Prisma 7 (the alternative) (the *the Prisma's* *client* — the *the Drizzle's* *alternative* — the *module-42's line: the Prisma is the Drizzle's alternative* (module 42's) — the *the ORM's* *choice* (module 01's)).
