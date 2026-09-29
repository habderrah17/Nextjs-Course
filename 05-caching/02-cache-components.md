# Module 21 — Cache Components Deep Dive: `'use cache'`, Keys & Variants

**Phase 5: Caching · Module 21 of 101**

> **Where does this run?** `'use cache'` is a `[SERVER]` directive. It marks a function/component as cacheable: its return value is stored by the framework, keyed by its inputs, and reused across requests (and across a render pass). Everything in this module is the *mechanics* of module 20's layer 3.

---

## 1. Concept — One directive, two levels, a compiler-generated key

`'use cache'` (inside a file with `cacheComponents: true`) does three things:

1. **Marks the scope cacheable.** It goes at the **top of a file** (all exports become cached — each must be `async`) or **at the top of an `async` function/component body** (that one function is cached). Synchronous functions cannot be cached (caching is about *async results*).
2. **Generates a cache key automatically.** The key = a serialized combination of:
   - **Build ID** (a deploy invalidates *everything* — the key includes the build, so new code = new keys)
   - **Function ID** (a secure hash of the function's location + signature in the codebase)
   - **Serializable arguments** (props for components, function args)
   - **Captured closure values** (if the cached function reads variables from the enclosing scope, they're bound as arguments — *this* is how "pass runtime data as args" becomes correctness: the arg is in the key)
   - (dev only) an HMR refresh hash (hot reload invalidates)
3. **Stores the output** in the cache handler (in-memory by default) and returns the stored value on subsequent calls with the same key — within a render pass *and* across requests, until revalidation.

**Data-level vs UI-level** (the two levels from module 20's WHAT question):

| Level | What's marked | What's stored | When to choose |
|---|---|---|---|
| **Data-level** | an async function (`getProducts()`) | its *return value* (DTOs) | the data is used by multiple UIs; you want to cache the *data* independently of how it's rendered |
| **UI-level** | a component/page (`BlogPosts()`) | its *rendered output* (serialized UI) | the whole fragment is the unit; internal data reads are private to it |

You can nest them (a UI-level component calling data-level functions) — the caching composes (module 22 covers nested behavior).

## 2. Mental Model — "the key is the contract"

The entire cache is a **map from key → output**. The key is auto-generated, so *you don't write keys — you write signatures*. This inverts the debugging:

- **Wrong stale data** = two *different* logical reads sharing a key → the missing dependency isn't an argument (it's a closure read the compiler couldn't capture, or a runtime API you illegally read inside) → the fix is to **promote the dependency to an argument**.
- **Wrong cache misses** (everything re-runs) = the key varies on something that shouldn't vary → you're passing `Date.now()` or an object with unstable identity as an arg → the fix is to **pass the stable discriminant** (the slug, the orgId), not the whole unstable thing.

**The runtime-API rule (enforced, not advisory):** `cookies()`, `headers()`, `searchParams` are *request data* — they cannot be read inside a `use cache` scope (build error). The pattern (module 16-03): read them in the *uncached* parent, pass the *values* into the cached function as arguments. The arguments join the key → per-user/per-request entries are *correct by construction*.

```tsx
// ❌ BUILD ERROR — request data inside a cache scope:
export async function getGreeting() {
  'use cache'
  const store = await cookies()          // ← not allowed
  return `Hi ${store.get('name')}`
}

// ✅ The pattern: read outside, pass in:
export default async function Page() {
  const store = await cookies()          // uncached scope
  const name = store.get('name')?.value
  return <Greeting name={name ?? undefined} />
}

async function Greeting({ name }: { name?: string }) {
  'use cache'
  cacheLife('hours')
  return <p>{name ? `Hi, ${name}` : 'Welcome'}</p>   // key includes `name` → per-user entries
}
```

## 3. Architecture — the two variants (and when each is the answer)

### 3.1 `'use cache: remote'` — durable, platform-backed

In-memory cache dies with the function instance (serverless: each instance has its own memory; a cold start = a cold cache). `'use cache: remote'` delegates the store to the **platform's cache handler** (a network roundtrip to check/store) — on Vercel this is the platform's durable cache; on self-hosted you supply `cacheHandlers` (module 22-05 area: Redis, etc.).

| | `'use cache'` (default) | `'use cache: remote'` |
|---|---|---|
| Where | process memory | platform/remote store |
| Shared across instances? | **no** | yes |
| Cost | ~free (memory) | network roundtrip + (on some platforms) cache fees |
| Use when | per-instance freshness is acceptable; hot data; low-cardinality keys | you need one cache across a fleet; long-lived data that must survive cold starts |

Rule: default to in-memory. Reach for remote when you've *measured* the per-instance cold-cache cost (serverless with many instances + high cardinality + expensive regeneration) — or when compliance/consistency demands a single source.

### 3.2 `'use cache: private'` — per-request isolation

The opposite problem: you *can't* refactor to pass runtime data as arguments (compliance constraints, deeply nested legacy code) — but you *must* not share entries across requests. `'use cache: private'` keeps the caching *within the request* (memoization semantics with cache-control semantics): the entry is scoped to the current request, never shared. It's the "I need the caching ergonomics without the cross-request sharing" escape hatch — and the docs' explicit guidance is: **refactor to args first; `private` is for when you genuinely can't.**

## 4. Production Code

### 4.1 Data-level: the catalog service (capstone)

`FILE: src/services/catalog.ts` (production pattern — [SERVER])

```ts
import 'server-only'
import { db } from '@/db'
import { products } from '@/db/schema'
import { desc, eq, ilike, and } from 'drizzle-orm'
import { cacheLife, cacheTag } from 'next/cache'
import type { ProductDTO } from '@/types/product'
import { productToDto } from './products'

/**
 * Cached catalog list — DATA-level cache.
 * Key = (buildId, functionId, { orgId, name, status, limit, cursor })
 * All five are serializable args → the key contract is the signature.
 */
export async function listCatalogProducts(args: {
  orgId: string
  name?: string
  status?: 'active' | 'draft' | 'archived'
  limit: number
  cursor?: string
}): Promise<{ items: ProductDTO[]; nextCursor: string | null }> {
  'use cache'
  cacheLife('hours')          // catalog freshness: hours (module 22)
  cacheTag('catalog')         // publish/delist invalidate (module 23)

  // (query as in module 17 — orgId-scoped, cursor-paginated)
  const rows = await db.select().from(products)
    .where(and(eq(products.orgId, args.orgId), eq(products.status, args.status ?? 'active'),
               args.name ? ilike(products.name, `%${args.name}%`) : undefined))
    .orderBy(desc(products.createdAt)).limit(args.limit + 1)
  const hasMore = rows.length > args.limit
  const items = rows.slice(0, args.limit).map(productToDto)
  return { items, nextCursor: hasMore ? items.at(-1)!.id : null }
}
```

### 4.2 UI-level: the pricing table (a page fragment worth caching as UI)

`FILE: src/components/pricing-table.tsx` (production pattern — [SERVER])

```tsx
import { cacheLife, cacheTag } from 'next/cache'
import { listPlans } from '@/services/plans'
import { PlansGrid } from '@/components/plans-grid'   // presentational, [BOTH]

/**
 * UI-level cache: the *rendered* pricing table is the unit.
 * Its internal read (listPlans) is private to this scope — no separate data cache entry.
 * (If other pages also need the plans as DATA, cache listPlans itself at data level instead —
 *  caching both the data and a UI that only wraps it would be double-work.)
 */
export async function PricingTable() {
  'use cache'
  cacheLife('days')           // pricing changes weekly at most
  cacheTag('pricing')         // the "update pricing" admin action invalidates

  const plans = await listPlans()
  return <PlansGrid plans={plans} />
}
```

### 4.3 File-level caching (the "whole module is cached" pattern)

`FILE: src/services/homepage-stats.ts` (simplified example — [SERVER])

```ts
'use cache'   // file-level: every export below is cached (each must be async)

import { db } from '@/db'
import { orders, orgs } from '@/db/schema'
import { sql } from 'drizzle-orm'

export async function getTotalSetups(): Promise<number> {
  // key = (buildId, functionId) — no args → ONE entry for the whole site.
  const [{ n }] = await db.select({ n: sql<number>`count(*)` }).from(orgs)
  return n
}

export async function getMonthlyRevenueWindow(month: string): Promise<number> {
  // key = (buildId, functionId, { month }) — per-month entries.
  const [{ n }] = await db.select({ n: sql<number>`sum(amount_cents)` }).from(orders).where(sql`${sql.month} = ${month}`)
  return n ?? 0
}
```

(Use file-level when the module is *coherently* a cache — a stats module, a CMS reader. Don't slap it on a file where only one of five functions should be cached: per-function directives are the more honest default.)

## 5. Common Mistakes

| Mistake | Symptom | Fix |
|---|---|---|
| Reading `cookies()`/`headers()`/`searchParams` inside `'use cache'` | Build error (by design) | Read outside, pass as args (the §2 pattern) |
| Caching a function that takes an unstable arg (`Date.now()`, a fresh object literal built each call) | Key churn → nothing ever hits | Pass the stable discriminant; build the object inside the cached function |
| Forgetting a closure value is *captured* (not re-read) | A cached function that "ignores" a config change until the build changes | The captured value is in the key — that's *correct* (the config is a dependency). If you wanted live config, it shouldn't be captured; pass it as an arg from an uncached caller |
| `use cache` on a *mutation* | Stale writes / silent no-ops | Mutations are never cached — direct DB + tag invalidation |
| Caching the same data at data level *and* inside a UI-level scope that re-reads it | Two entries, two invalidations to keep in sync | One level: the data function (shared) or the UI (private) — not both |
| Expecting in-memory entries to survive a deploy | "Cache went cold after every deploy" — true, and expected (build ID in the key) | That's the safety property (new code, new data). For durability across deploys you *want* stale: `cacheLife('max')` + platform/remote cache |
| `'use cache: remote'` everywhere "for consistency" | Roundtrip + fees on every read | Remote only where cross-instance sharing is measured-justified |

## 6. Security Notes

- The key-is-the-contract model is the anti-bleed control (module 20): if user A's entry could serve user B, a dependency is missing from the key — and the framework would have *stopped you* reading it inside the scope. The remaining risk is your *design*: an arg named `id` that the caller fills from the *wrong* source (client input instead of session) is a key that's complete but *wrong* — the tenancy rules of module 11 apply to cache args exactly as to query args.
- `cacheTag` values are *public within your app* but the cached *content* is not: a tag name leak tells an attacker what to target, not what you store. Don't encode PII in tags (`user:{email}` → `user:{userId}`; better: tag by *resource type*, not identity).
- Remote caches (Redis) hold your DTOs: review them in the data-retention/encryption pass (module 19).

## 7. Performance Notes

- Data-level caching is the **deduplicator**: N components reading `getExchangeRates()` in one page = 1 execution (L4) and 1 entry (L3) shared across pages. Design your services so shared data has *one* cached home.
- UI-level caching stores *serialized UI* — bigger than a DTO, but it skips the *render* on hit. Use it for expensive-to-render fragments (big tables, generated lists), not for tiny values (cache the value instead).
- The dev overlay shows cache hits/misses and "blocking-route" insights (uncached reads that could be streamed) — treat overlay insights as a *perf lint* (module 18-01).

## 8. Exercise

**Beginner.** Build the §4.1 catalog function in your scaffold. In dev: (a) two requests, same args → one DB execution (log inside); (b) different `name` → separate entries; (c) a deploy (`next build`) → new build ID → cold. Screenshot the logs for each.

**Intermediate.** Take an uncached page that reads `cookies()` for a theme and a profile name. Refactor to the §2 pattern (read outside, pass in, UI-level cache the greeting). Prove: two users (two cookie values) get separate entries (log the key's arg values), and the *rest* of the page still prerenders (the dev overlay stops flagging it).

**Production.** Implement the pricing table at UI level with `cacheTag('pricing')` and write the admin "update plan" action that calls `updateTag('pricing')` + `redirect`. Verify: (a) the public page serves the shell from cache (no DB on hit), (b) after the admin update, the *next* public render is fresh (the actor sees it immediately — read-your-own-writes), (c) two "users" hitting the page concurrently during the first regeneration don't stampede (observe: one execution, both get the result — the framework dedupes regeneration).

## 9. Architecture Challenge

**Prompt:** You have `getTeamProfile(teamId)` (a join of 4 tables, 300ms cold) used by: (1) the team's public page (10k teams, long-tail traffic), (2) the admin "edit team" page (needs fresh data every open), (3) a "featured teams" carousel on the landing page (8 teams, changes weekly).

Design the caching: how many `use cache` functions do you write (data-level? UI-level? both?), what args each takes (think: the admin page's freshness requirement vs the public page's), what tags, and what the landing page's carousel *doesn't* cache (hint: it's not the profile — it's the *selection* of 8).

<details>
<summary>Model answer</summary>
One **data-level** `getTeamProfileCached(teamId)`: `'use cache'`, `cacheLife('hours')`, `cacheTag('team:' + teamId)` (fine-grained tag — editing team X shouldn't cool 10k other entries… with `revalidateTag('team:'+id)`; a coarser 'teams' tag for the carousel's sake). Key includes `teamId` → per-team entries; the long tail means each entry is filled by its own traffic (no build-time generation for 10k teams — that's the App-Shell pattern, module 07: known/head teams prerender, the tail streams).
Admin edit page: does **not** call the cached function — it calls `getTeamProfile` (uncached) — the actor must see current data *before* editing (a 1-hour-stale profile on an edit screen is a data-entry hazard). After the admin *saves*, the action calls `updateTag('team:'+teamId)` (immediate — read-your-own-writes for the admin) — the public view follows on its next hit.
Landing carousel: the *selection* ("which 8 teams are featured") is its own tiny cached read: `getFeaturedTeamIds()` — `cacheLife('weeks')`, `cacheTag('featured-teams')`, invalidated by the weekly curation job (or the admin's "set featured" action). The carousel then maps those 8 ids through `getTeamProfileCached` (shared entries). Caching "the carousel" as UI would re-embed 8 full profiles into a UI blob and re-render them weekly — caching the *selection* (8 ids) + the *profiles* (shared data entries) is the composable version: the curation job invalidates one small entry, not a big one.
Three functions, two tags + one coarse tag, zero stampedes, and the admin's freshness contract is a *different function*, not a different life on the same one.
</details>

## 10. Official Documentation

- `use cache`: https://nextjs.org/docs/app/api-reference/directives/use-cache
- `use cache: remote`: https://nextjs.org/docs/app/api-reference/directives/use-cache-remote
- `use cache: private`: https://nextjs.org/docs/app/api-reference/directives/use-cache-private
- Caching (Cache Components): https://nextjs.org/docs/app/getting-started/caching
- Cache Handlers: https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheHandlers
- cacheComponents config: https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents

## 11. What You Should Know Before Continuing

- [ ] I can state what `'use cache'` does (key generation, storage, reuse) and where the key comes from
- [ ] I can choose data-level vs UI-level and explain the nesting rules
- [ ] I apply the runtime-API pattern (read outside, pass in) and can explain why it's the *security* fix
- [ ] I know when `remote` and `private` are the right answer (and that `private` means "you should have refactored")
- [ ] I've verified deduped regeneration (one execution, many readers) in dev

**Next:** Module 22 — `cacheLife`: the profiles, the custom config, nested caching, and what "stale" actually buys you.
