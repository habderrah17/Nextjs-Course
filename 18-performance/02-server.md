# Module 70 — Server Performance: TTFB, Waterfalls, Cache Hits, Connection Reuse

**Phase 18: Performance · Module 70 of 101**

> **Where does this run?** Everything in this module is **`[SERVER]`** (the TTFB is the server's clock, module 69's; the waterfalls are the server's awaits, module 70's §1; the cache hits are the Cache Components' tags, module 20's; the connection reuse is the pool's, module 70's §4). The module-70's standing rule (module 20's 3 clocks + module 37's tenancy, now the server level): **TTFB is the sum of the server's sequential awaits — parallelize what is independent (`Promise.all`), cache what is repeatable (the `cacheTag`, module 20's), and reuse the connection (the pool, module 37's) (module 70's §1)** (module 70's §1).

---

## 1. Concept — The TTFB is the server's clock (the 4 levers)

**The TTFB's** (module 70's §1.1): the *the server's total's* (module 70's §1.1) — the *module-70's line: the TTFB is the 800ms's* (module 70's §1.1) — the *the no client's* (module 70's §1.1).

**The waterfall's** (module 70's §1.2): the *the sequential's `await`'s* (module 70's §1.2) — the *module-70's line: the waterfall is the `Promise.all`'s* (module 70's §1.2) — the *the no 5's awaits* (module 70's §1.2).

**The cache hit's** (module 70's §1.3): the *the `cacheTag`'s* (module 20's) — the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3) — the *module-20's line: the tag is the typed's* (module 20's).

**The connection's** (module 70's §1.4): the *the pool's* (module 37's) — the *module-70's line: the connection is the PgBouncer's* (module 37's) — the *the no per-request's* (module 70's §1.4).

## 2. Mental Model — The TTFB's breakdown (drawn)

```mermaid
flowchart TD
    A["THE REQUEST (module 70's §1) — the the TTFB's (module 70's §1.1) — the the 800ms's (module 70's §1.1)"] --> B["THE SESSION (module 43's) — the the use cache: private (module 47's)"]
    B --> C["THE WATERFALL (module 70's §1.2) — the the 5's awaits (module 70's §1.2)"]
    C --> C1["THE stats' query (module 68's §1.2) — the the 1's query (module 68's §1.2)"]
    C --> C2["THE chart's data (module 68's §1.3) — the the series' (module 68's §1.3)"]
    C --> C3["THE recent's (module 68's §3.3) — the the 10's rows (module 68's §3.3)"]
    C1 --> D["THE PARALLEL (module 70's §1.2) — the the Promise.all (module 70's §1.2) — the the no 5's awaits (module 70's §1.2)"]
    C2 --> D
    C3 --> D
    D --> E["THE RENDER (module 70's §1) — the the RSC's (module 70's §1) — the the 50ms's (module 70's §1.1)"]
    E --> F["THE TTFB (module 70's §1.1) — the the 350ms's (module 70's §1.1)"]
    G["THE CACHE (module 70's §1.3) — the the cacheTag's (module 20's) — the the 0ms's (module 70's §1.3)"] --> C1
    G --> C2
    G --> C3
```

**The TTFB's breakdown** (the module-70's mental model):
1. **The TTFB** (module 70's §1.1): the *the 800ms's* — the *module-70's line: the TTFB is the 800ms's* (module 70's §1.1).
2. **The waterfall** (module 70's §1.2): the *the `Promise.all`'s* — the *module-70's line: the waterfall is the `Promise.all`'s* (module 70's §1.2).
3. **The cache hit** (module 70's §1.3): the *the 0ms's* — the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3).
4. **The connection** (module 70's §1.4): the *the PgBouncer's* — the *module-70's line: the connection is the PgBouncer's* (module 37's).

## 3. Architecture — The 4 levers (the code)

### 3.1 The TTFB's (module 70's §1.1 — the measure's)

`FILE: src/middleware.ts` (production pattern — [SERVER] — the module-70's §3.1: the `x-ttfb`'s)

```ts
// THE TTFB'S (module 70's §3.1) — the the x-ttfb's (module 70's §1.1) — the the [SERVER] (module 70's §1):
// 'use edge'
import { NextResponse, type NextRequest } from 'next/server'

export function middleware(req: NextRequest) {
  const start = Date.now()   /* the module-70's line: the start is the request's (module 70's §3.1) */
  const res = NextResponse.next()
  res.headers.set('x-ttfb', String(Date.now() - start))   /* the module-70's line: the x-ttfb is the measure's (module 70's §1.1) */
  return res
}
/* THE RULE (module 70's §3.1): the the TTFB is the 800ms's (module 70's §1.1) — the the x-ttfb is the measure's (module 70's §1.1) */
```

**The module-70's line:** the *TTFB is the 800ms's* (module 70's §1.1) — the *`x-ttfb` is the measure's* (module 70's §3.1).

### 3.2 The waterfall's (module 70's §1.2 — the `Promise.all`'s)

`FILE: src/services/dashboard.ts` (production pattern — [SERVER] — the module-70's §3.2: the parallel's)

```ts
// THE WATERFALL'S (module 70's §1.2) — the the no 5's awaits (module 70's §1.2) — the the Promise.all (module 70's §1.2):
import { db } from '@/db'
import { products, orders, revenue } from '@/db/schema'   /* the module-37's line: the table is the orgId's FK (module 37's) */
import { and, eq, gte, desc, sql } from 'drizzle-orm'

export async function getDashboardStats(orgId: string, range: string) {
  /* THE WRONG (module 70's §3.2) — the the 5's awaits (module 70's §1.2):
     const revenue = await getRevenue(orgId, range)   (module 70's §3.2) — the the 200ms (module 70's §1.2)
     const orders = await getOrders(orgId, range)   (module 70's §3.2) — the the 150ms (module 70's §1.2)
     const products = await getProducts(orgId, range)   (module 70's §3.2) — the the 100ms (module 70's §1.2)
     const avg = await getAvg(orgId, range)   (module 70's §3.2) — the the 80ms (module 70's §1.2)
     const recent = await getRecent(orgId)   (module 70's §3.2) — the the 50ms (module 70's §1.2)
     /* TOTAL: 580ms (module 70's §1.2) — the the no Promise.all (module 70's §1.2) */

  /* THE RIGHT (module 70's §3.2) — the the Promise.all (module 70's §1.2): */
  const [revenue, orders, products, avg, recent] = await Promise.all([
    getRevenue(orgId, range),   /* the module-70's line: the parallel's (module 70's §1.2) — the the 200ms (module 70's §1.2) */
    getOrders(orgId, range),   /* the module-70's line: the parallel's (module 70's §1.2) — the the 150ms (module 70's §1.2) */
    getProducts(orgId, range),   /* the module-70's line: the parallel's (module 70's §1.2) — the the 100ms (module 70's §1.2) */
    getAvg(orgId, range),   /* the module-70's line: the parallel's (module 70's §1.2) — the the 80ms (module 70's §1.2) */
    getRecent(orgId),   /* the module-70's line: the parallel's (module 70's §1.2) — the the 50ms (module 70's §1.2) */
  ])
  /* TOTAL: 200ms (module 70's §1.2) — the the no 580ms (module 70's §1.2) */
  return { revenue, orders, products, avgOrderValue: avg, series: revenue.series, recent }
}
```

**The module-70's line:** the *waterfall is the `Promise.all`'s* (module 70's §1.2) — the *the no 5's awaits* (module 70's §1.2) — the *the 200ms is the parallel's* (module 70's §1.2).

### 3.3 The cache hit's (module 70's §1.3 — the `cacheTag`'s)

`FILE: src/services/dashboard.ts` (production pattern — [SERVER] — the module-70's §3.3: the `use cache`'s)

```ts
// THE CACHE HIT'S (module 70's §1.3) — the the cacheTag's (module 20's) — the the use cache (module 20's):
import { cache, cacheTag, cacheLife } from 'react'   /* the module-20's line: the cache's (module 20's) */

const getRevenueCached = cache(async (orgId: string, range: string) => {
  cacheLife('max', 3600)   /* the module-20's line: the cacheLife is the 1h's (module 20's) */
  cacheTag('revenue:' + orgId + ':' + range)   /* the module-20's line: the tag is the typed's (module 20's) — the the no PII (module 20's) */
  return getRevenue(orgId, range)   /* the module-70's line: the no DB's on the hit's (module 70's §1.3) */
})

export async function getDashboardStatsCached(orgId: string, range: string) {
  return getRevenueCached(orgId, range)   /* the module-70's line: the 0ms's on the hit's (module 70's §1.3) */
}
/* THE RULE (module 70's §3.3): the the tag is the typed's (module 20's) — the the 0ms's on the hit's (module 70's §1.3) — the the revalidateTag's (module 20's) */
```

**The module-70's line:** the *cache hit is the 0ms's* (module 70's §1.3) — the *tag is the typed's* (module 20's) — the *`cacheLife` is the 1h's* (module 20's).

### 3.4 The connection's (module 70's §1.4 — the PgBouncer's)

`FILE: .env` + `FILE: src/db/index.ts` (production pattern — [SERVER] — the module-70's §3.4: the pool's)

```ts
// THE CONNECTION'S (module 70's §3.4) — the the PgBouncer's (module 37's) — the the no per-request's (module 70's §1.4):
// DATABASE_URL=postgres://app:pass@pgbouncer.internal:6432/shop   /* the module-70's line: the pgbouncer's port is the 6432's (module 70's §3.4) */
import { drizzle } from 'drizzle-orm/postgres-js'
import postgres from 'postgres'

const sql = postgres(process.env.DATABASE_URL!, {
  max: 20,   /* the module-70's line: the max is the 20's (module 70's §3.4) — the the no 100's (module 70's §1.4) */
  idleTimeout: 30,   /* the module-70's line: the idleTimeout is the 30s's (module 70's §3.4) */
  connect_timeout: 10,   /* the module-70's line: the connect_timeout is the 10s's (module 70's §3.4) */
})
export const db = drizzle(sql)
/* THE RULE (module 70's §3.4): the the max is the 20's (module 70's §3.4) — the the PgBouncer's is the 6432's (module 70's §3.4) — the the no per-request's (module 70's §1.4) */
```

**The module-70's line:** the *connection is the PgBouncer's* (module 37's) — the *max is the 20's* (module 70's §3.4) — the *no per-request's* (module 70's §1.4).

## 4. Production Code — The cache hit's log (module 70's §4)

`FILE: src/instrumentation.node.ts` (production pattern — [SERVER] — the module-70's §4: the hit's rate)

```ts
// THE CACHE HIT'S LOG (module 70's §4) — the the hit's rate (module 70's §4) — the the [SERVER] (module 70's §1):
let cacheHits = 0
let cacheMisses = 0

export function logCacheHit() { cacheHits++ }   /* the module-70's line: the hit's is the 0ms's (module 70's §1.3) */
export function logCacheMiss() { cacheMisses++ }   /* the module-70's line: the miss's is the DB's (module 70's §1.3) */

export function getCacheHitRate() {
  const total = cacheHits + cacheMisses
  return total === 0 ? 1 : cacheHits / total   /* the module-70's line: the hit's rate is the 90%'s (module 70's §4) */
}
/* THE RULE (module 70's §4): the the hit's rate is the 90%'s (module 70's §4) — the the no 100%'s (module 70's §4) — the the no 0%'s (module 70's §4) */
```

**The module-70's line:** the *hit's rate is the 90%'s* (module 70's §4) — the *the no 100%'s* (module 70's §4) — the *the no 0%'s* (module 70's §4).

## 5. Common Mistakes (the server's failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The 5's awaits** (module 70's §1.2's line violated) | the *module-70's line: the waterfall is the `Promise.all`'s* (module 70's §1.2) — the *the 5's awaits' is the *no's* (module 70's §1.2) — the *module-70's line: the no 5's awaits* (module 70's §1.2) — the *no 5's awaits* (module 70's §1.2)* | the *the `Promise.all`'s (module 70's §1.2) — the *module-70's line: the waterfall is the `Promise.all`'s* (module 70's §1.2)* |
| **The no `cacheTag`** (module 70's §1.3's line violated) | the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3) — the *the no `cacheTag`'s is the *no's* (module 70's §1.3) — the *module-70's line: the no `cacheTag`'s* (module 70's §1.3) — the *no `cacheTag`'s* (module 70's §1.3)* | the *the `cacheTag`'s (module 20's) — the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3)* |
| **The no pool** (module 70's §1.4's line violated) | the *module-70's line: the connection is the PgBouncer's* (module 37's) — the *the no pool's is the *no's* (module 70's §1.4) — the *module-70's line: the no pool's* (module 70's §1.4) — the *no pool's* (module 70's §1.4)* | the *the `postgres`'s `max: 20` (module 70's §3.4) — the *module-70's line: the connection is the PgBouncer's* (module 37's)* |
| **The 100's max** (module 70's §3.4's line violated) | the *module-70's line: the max is the 20's* (module 70's §3.4) — the *the 100's max's is the *no's* (module 70's §3.4) — the *module-70's line: the no 100's* (module 70's §1.4) — the *no 100's* (module 70's §1.4)* | the *the `max: 20`'s (module 70's §3.4) — the *module-70's line: the max is the 20's* (module 70's §3.4)* |
| **The 0%'s hit rate** (module 70's §4's line violated) | the *module-70's line: the hit's rate is the 90%'s* (module 70's §4) — the *the 0%'s hit rate's is the *no's* (module 70's §4) — the *module-70's line: the no 0%'s* (module 70's §4) — the *no 0%'s* (module 70's §4)* | the *the `cacheTag`'s + the `revalidateTag`'s (module 20's) — the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3)* |
| **The no `x-ttfb`** (module 70's §3.1's line violated) | the *module-70's line: the TTFB is the 800ms's* (module 70's §1.1) — the *the no `x-ttfb`'s is the *no's* (module 70's §3.1) — the *module-70's line: the no `x-ttfb`'s* (module 70's §3.1) — the *no `x-ttfb`'s* (module 70's §3.1)* | the *the `middleware`'s `x-ttfb` (module 70's §3.1) — the *module-70's line: the TTFB is the 800ms's* (module 70's §1.1)* |

## 6. Security Notes

- **The no PII in the tag** (module 20's): the *module-20's line: the tag is the no PII's* (module 20's) — the *module-75's* *deep-dive* (module 75's).
- **The pool's** (module 70's §3.4): the *module-70's line: the max is the 20's* (module 70's §3.4) — the *module-75's* *deep-dive* (module 75's).

## 7. Performance Notes

- **The `Promise.all`'s** (module 70's §1.2): the *module-70's line: the waterfall is the `Promise.all`'s* (module 70's §1.2) — the *the 200ms is the parallel's* (module 70's §1.2).
- **The cache hit's** (module 70's §1.3): the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3) — the *the no DB's on the hit's* (module 70's §1.3).
- **The pool's** (module 70's §3.4): the *module-70's line: the max is the 20's* (module 70's §3.4) — the *the no per-request's* (module 70's §1.4).

## 8. Exercise

**Beginner.** *The `x-ttfb`'s* (module 70's §3.1): the *the `middleware`'s* (module 3.1's) + the *the `x-ttfb`'s* (module 3.1's) — *build it* — the *artifact: the TTFB's* (module 3.1's).

**Intermediate.** *The `Promise.all`'s* (module 70's §3.2): the *the 5's queries' parallel* (module 3.2's) + the *the 200ms's* (module 3.2's) — *build it* — the *artifact: the parallel's* (module 3.2's).

**Production.** *The `cacheTag`'s + the pool's* (module 70's §3.3 + §3.4): the *the `use cache`'s* (module 3.3's) + the *the `postgres`'s `max: 20`* (module 3.4's) + the *the hit's rate's* (module 4's) — *build it* — the *artifact: the cache's* (module 3.3's).

## 9. Architecture Challenge

**Prompt:** The *"the team's dashboard takes 1.8s to TTFB, has 5 sequential awaits, and no cache"* (the *module-70's* *server* — the *module-20's* *cache* — the *module-70's line: the TTFB is the 800ms's* (module 70's §1.1) — the *module-20's line: the tag is the typed's* (module 20's) — the *module-70's standing line: the TTFB is the 800ms's + the waterfall is the `Promise.all`'s + the cache hit is the 0ms's* (module 70's §1.1 + module 70's §1.2 + module 70's §1.3)).

The *problems*: (1) the *the 5's awaits* (the *the no `Promise.all`'s* (module 70's §1.2) — the *module-70's line: the waterfall is the `Promise.all`'s* (module 70's §1.2) — the *module-70's standing line: the waterfall is the `Promise.all`'s* (module 70's §1.2)).

(2) the *the no cache* (the *the no `cacheTag`'s* (module 20's) — the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3) — the *module-70's standing line: the cache hit is the 0ms's* (module 70's §1.3)).

**Design**: the *the server's remediation* (the *the `Promise.all`'s* (module 70's §1.2) + the *the `cacheTag`'s* (module 20's) + the *the pool's* (module 37's) — the *module-70's line: the TTFB is the 800ms's* (module 70's §1.1) — the *module-70's standing line: the TTFB is the 800ms's + the waterfall is the `Promise.all`'s + the cache hit is the 0ms's* (module 70's §1.1 + module 70's §1.2 + module 70's §1.3)).

Produce: the *the server's remediation* (the *the `Promise.all`'s* (module 70's §1.2) + the *the `cacheTag`'s* (module 20's) + the *the pool's* (module 37's) — the *module-70's line: the TTFB is the 800ms's* (module 70's §1.1) — the *module-70's standing line: the TTFB is the 800ms's + the waterfall is the `Promise.all`'s + the cache hit is the 0ms's* (module 70's §1.1 + module 70's §1.2 + module 70's §1.3)).

<details>
<summary>Model answer</summary>
**The server's remediation** (module 70's §1.2 + module 20's + module 37's):
1. **The `Promise.all`'s** (module 70's §1.2): the *the 5's awaits become the parallel's* — the *module-70's line: the waterfall is the `Promise.all`'s* (module 70's §1.2).
2. **The `cacheTag`'s** (module 20's): the *the `use cache` + the `cacheTag` is the 0ms's* — the *module-70's line: the cache hit is the 0ms's* (module 70's §1.3).
3. **The pool's** (module 37's): the *the `max: 20` + the PgBouncer is the connection's* — the *module-70's line: the connection is the PgBouncer's* (module 37's).
**The generalization** (the *server's* pattern, the *module's* standing rule): **the *TTFB is the 800ms's* (module 70's §1.1) — the *the waterfall is the `Promise.all`'s* (module 70's §1.2) — the *the cache hit is the 0ms's* (module 70's §1.3) — the *module-70's standing line: the TTFB is the 800ms's + the waterfall is the `Promise.all`'s + the cache hit is the 0ms's* (module 70's §1.1 + module 70's §1.2 + module 70's §1.3)*.
</details>

## 10. Official Documentation

- Next.js: Cache Components: https://nextjs.org/docs/app/guides/caching-with-cache-components
- Next.js: `revalidateTag`: https://nextjs.org/docs/app/api-reference/functions/revalidate-tag
- web.dev: TTFB: https://web.dev/articles/ttfb
- Postgres: `postgres` (postgres-js): https://github.com/porsager/postgres
- PgBouncer: https://www.pgbouncer.org/
- The module-20's cache: the module-20 (the phase-4's file-04)

## 11. What You Should Know Before Continuing

- [ ] I can state the *4 levers* (module 1's: the TTFB/waterfall/cache/connection) — the *module-70's line: the TTFB is the 800ms's* (module 1's)
- [ ] I know the *TTFB is the 800ms's* (module 1.1's) — the *the `x-ttfb`'s* (module 3.1's)
- [ ] I know the *waterfall is the `Promise.all`'s* (module 1.2's) — the *the no 5's awaits* (module 1.2's)
- [ ] I know the *cache hit is the 0ms's* (module 1.3's) — the *the `cacheTag`'s* (module 20's)
- [ ] I know the *connection is the PgBouncer's* (module 1.4's) — the *the max is the 20's* (module 3.4's)
- [ ] I know the *hit's rate is the 90%'s* (module 4's) — the *the no 0%'s* (module 4's)
- [ ] I've done the *`x-ttfb`'s* (module 8's beginner) + the *`Promise.all`'s* (module 8's intermediate) + the *`cacheTag`/pool* (module 8's production) — the *artifacts* (module 20's)

**Next:** Module 71 — Client Performance (the *the bundle's budget* — the *the hydration's* — the *module-71's line: the client is the 200KB's* (module 71's)).
