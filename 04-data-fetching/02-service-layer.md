# Module 17 — The Service Layer: The Seam Between Components and the Database

**Phase 4: Data Fetching · Module 17 of 101**

> **Where does this run?** `[SERVER]` — services are the *only* layer that imports `db/`, and they are imported *only* by server code. This is the structural decision that makes the rest of the course (authz, caching, multi-tenancy, testing) tractable.

---

## 1. Concept — Why a seam at all?

You *could* write Drizzle queries directly in pages and Server Functions. You would, for a weekend app. At the capstone's size, that fails in five specific ways:

1. **Tenancy**: `orgId` scoping must appear in *every* query. Scattered inline = one forgotten `where` = cross-tenant read. One layer = one place to enforce (module 11-02).
2. **Caching**: `use cache` + tags belong on *functions with stable identity*. Query-in-page means the cache key is "this page render"; service means the cache key is "getProduct(slug)" — composable across pages (module 05-02).
3. **Authorization**: permission checks ("can this user cancel this order?") are *domain logic*, not rendering. Pages shouldn't contain policy (module 11-01).
4. **DTO mapping**: the ORM row → DTO conversion (module 14) should happen once, at one layer, with the compiler enforcing completeness.
5. **Testing**: services are plain async functions → unit/integration testable with a real DB and zero HTTP, zero rendering (module 20-02).

**The service is the only place that knows SQL.** Pages and actions know *domain operations* ("list products for the org with this filter"); services know *how* (joins, cursors, indexes).

## 2. Mental Model — the layer contract

```
┌────────────────────────────────────────────────────────────────┐
│ PAGES / LAYOUTS / SERVER FUNCTIONS / ROUTE HANDLERS            │
│   call:  await listProducts(orgId, input)                      │
│   know:  the operation, the DTO, the error contract            │
│   don't: import db, write where-clauses, know column names     │
├────────────────────────────────────────────────────────────────┤
│ SERVICES  (services/*.ts)                                       │
│   do:    ORM queries, DTO mapping, tenancy scoping,            │
│          permission checks, caching ('use cache' + tags)       │
│   know:  the schema, the index strategy, the error taxonomy    │
├────────────────────────────────────────────────────────────────┤
│ DB  (db/)  — schema.ts (Drizzle), client with server-only guard│
└────────────────────────────────────────────────────────────────┘
```

**The contract, as code-review rules:**

1. `services/*` may import `db/*`, `types/*`, `schemas/*`, `lib/*` (server parts). Nothing else.
2. `services/*` exports **async functions only** (or pure mappers marked as such). No classes, no singletons with state.
3. Every service that reads tenant data takes `orgId` as its **first parameter** (from the session — the *caller* resolves it, the service *requires* it).
4. Every service returns **DTOs** (or `Page<DTO>`) — never `typeof ….$inferSelect`.
5. Every service that can fail with a *domain* error throws a **typed domain error** (module 04-03's `ApiError` family) — pages/actions translate it to UI/HTTP.

## 3. Architecture — a service, fully dressed

`FILE: src/services/products.ts` (production pattern — [SERVER])

```ts
import 'server-only'   // build-time guard: importing this from a client file = build error (module 09-01)

import { db } from '@/db'
import { products } from '@/db/schema'
import { and, asc, desc, eq, ilike, lt, or, sql } from 'drizzle-orm'
import { cacheLife, cacheTag } from 'next/cache'
import { z } from 'zod'
import type { ProductCreateInput, ProductDTO, ProductFilter, ProductPatchInput } from '@/features/product/product-schemas'
import type { Page } from './typed-list'
import { AppError } from '@/lib/errors'
import { assertOrgMembership } from './authorization'

// ---------- READS ----------

export async function listProducts(orgId: string, input: { filter: ProductFilter; sort: { by: string; dir: 'asc' | 'desc' }; limit: number; cursor?: string }): Promise<Page<ProductDTO>> {
  const clauses = [eq(products.orgId, orgId)]                    // ① tenancy — first, always
  if (input.filter.name) clauses.push(ilike(products.name, `%${input.filter.name}%`))
  if (input.filter.status) clauses.push(eq(products.status, input.filter.status))
  if (input.cursor) clauses.push(lt(products.id, input.cursor))  // ② cursor pagination (module 09-04)

  const sortCol = input.sort.by === 'price_asc' ? products.priceCents : input.sort.by === 'price_desc' ? products.priceCents : products.createdAt
  const direction = (input.sort.by === 'price_asc' || input.sort.by === 'newest') && input.sort.dir === 'asc' ? asc : desc

  const rows = await db.select().from(products).where(and(...clauses)).orderBy(direction(sortCol), desc(products.id)).limit(input.limit + 1)
  const hasMore = rows.length > input.limit
  const items = rows.slice(0, input.limit).map(productToDto)
  const [{ n }] = await db.select({ n: sql<number>`count(*)` }).from(products).where(and(eq(products.orgId, orgId)))
  return { items, nextCursor: hasMore ? items.at(-1)!.id : null, total: n }
}

/** Cached read — used by public catalog & detail pages (module 05-02). */
export async function getProductBySlugCached(slug: string): Promise<ProductDTO | null> {
  'use cache'
  cacheLife('hours')                       // catalog content: changes on publish
  cacheTag('products')                     // publish/delist actions revalidate this (module 05-04)
  return getProductBySlug(slug)
}

export async function getProductBySlug(slug: string): Promise<ProductDTO | null> {
  const row = await db.select().from(products).where(eq(products.slug, slug)).limit(1)
  return row[0] ? productToDto(row[0]) : null
}

// ---------- WRITES ----------

export async function createProduct(orgId: string, input: ProductCreateInput, actorId: string): Promise<ProductDTO> {
  await assertOrgMembership(actorId, orgId)                       // ③ authz at the service (module 11-01)
  const existing = await db.select().from(products).where(eq(products.slug, input.slug)).limit(1)
  if (existing.length) throw new AppError({ status: 409, code: 'conflict', message: 'Slug already taken' })

  const [row] = await db.insert(products).values({
    orgId,
    name: input.name,
    slug: input.slug,
    description: input.description,
    priceCents: input.priceCents,
    status: 'draft',
    images: [],
    specs: input.specs ?? null,
  }).returning()

  return productToDto(row)
}

export async function updateProduct(orgId: string, id: string, input: ProductPatchInput, actorId: string): Promise<ProductDTO> {
  await assertOrgMembership(actorId, orgId)
  // Fetch-scoped update: the WHERE clause enforces tenancy — a foreign ID is a 404, not a silent 0-row update.
  const [row] = await db.update(products).set({ ...input, updatedAt: new Date() })
    .where(and(eq(products.id, id), eq(products.orgId, orgId))).returning()
  if (!row) throw new AppError({ status: 404, code: 'not_found', message: 'Product not found' })
  return productToDto(row)
}

// ---------- MAPPER ----------

function productToDto(p: typeof products.$inferSelect): ProductDTO {
  return {
    id: p.id, name: p.name, slug: p.slug, description: p.description,
    priceCents: p.priceCents, priceFormatted: formatCents(p.priceCents),
    status: p.status, imageUrls: p.images, specs: p.specs,
    updatedAt: p.updatedAt.toISOString(),
  }
}
```

Every numbered comment is a contract rule in action: ① tenancy first, ② cursor pagination, ③ authz at the seam, plus caching on the *stable* read, DTOs out, typed errors thrown.

## 4. Production Code — the `AppError` taxonomy (shared by pages, actions, routes)

`FILE: src/lib/errors.ts` (production pattern — [SERVER])

```ts
// One error family for the whole server. Pages/actions translate to UI,
// route handlers translate to HTTP. Nothing else is thrown across the layer.
export class AppError extends Error {
  readonly status: number
  readonly code: string
  constructor(public readonly body: { status: 400 | 401 | 403 | 404 | 409 | 422 | 500; code: 'validation' | 'unauthenticated' | 'forbidden' | 'not_found' | 'conflict' | 'business_rule' | 'internal'; message: string; fieldErrors?: Record<string, string[]> }) {
    super(body.message)
    this.status = body.status
    this.code = body.code
  }
}

export function isAppError(e: unknown): e is AppError {
  return e instanceof AppError
}

// Unexpected errors (DB down, bugs) are NOT AppErrors — they 500 with logging (module 21-01),
// never with their message to the user.
```

`FILE: src/features/product/product-actions.ts` (excerpt — [SERVER], how an action consumes the service)

```ts
'use server'
import { headers } from 'next/headers'
import { auth } from '@/lib/auth'
import { updateTag } from 'next/cache'
import { z } from 'zod'
import { createProduct, listProducts } from '@/services/products'
import { AppError, isAppError } from '@/lib/errors'
import { productFormSchema } from './product-schemas'

export async function createProductAction(formData: FormData) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })

  const parsed = productFormSchema.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    // Field errors round-trip to the form (module 12-03):
    const fieldErrors: Record<string, string[]> = {}
    for (const issue of parsed.error.issues) fieldErrors[String(issue.path[0] ?? '_form')] = [issue.message]
    throw new AppError({ status: 400, code: 'validation', message: 'Fix the highlighted fields', fieldErrors })
  }

  await createProduct(session.organizationId, parsed.data, session.user.id)
  updateTag('products')                       // read-your-own-writes
  // redirect('/products/manage?created=1')  — or let the client's transition refresh
}
```

## 5. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| Queries in pages "for now" | Tenancy/orgId enforcement scatters; no stable cache keys; untestable renders | The service is the *first* abstraction you make, not the last — module 01-04's "folders born with their first citizen" |
| Service takes `userId` and resolves org internally "for convenience" | You've hidden the tenancy decision; every caller must trust the service's org resolution | The *caller* (with the session) resolves `orgId`; the service *requires* it — the audit trail shows who decided what |
| Service returns ORM rows | DTO contract broken; schema changes ripple to the client | `toDto` at the exit; the compiler enforces it |
| `try/catch` around every service call in pages | Exception soup; errors get swallowed inconsistently | Throw `AppError`; *one* translation point per surface (page renders, action re-throws to the form, route maps to HTTP) |
| Caching a *write* path | Mutations must never be cached — `use cache` on a mutation = stale writes | Only reads are cached; writes call the DB directly and then invalidate tags |
| A "god service" (products.ts = 1,200 lines, 30 functions) | Untestable, unreviewable | Split by *operation group* when a file passes ~400 lines; keep the module contract the same |

## 6. Security Notes

- The service is **the authorization boundary for data** (module 11-01): `assertOrgMembership` + resource-scoped WHERE clauses. The UI hiding a button is *not* this.
- `import 'server-only'` in every `db/` file (module 09-01) makes "client imports the service" a **build error** — the structural guarantee the security model depends on.
- Error messages: `AppError.message` is user-facing by contract — never put SQL, IDs of *other* resources, or stack fragments in it.

## 7. Performance Notes

- Stable service identity = stable cache keys = composable caching (two pages calling `getProductBySlugCached(slug)` share one entry — the data cache is a *deduplicator*).
- N+1 is a *service-design* failure (module 18-04): `listProducts` that then loops `await getImages(p.id)` per product. The service is where you fix it (batch join / `inArray`), which is another reason queries can't live in pages.
- `count(*)` for pagination totals: index-friendly (module 09-04) — and *optional* for cursor pagination (the cursor doesn't need a total; render "Next" from `nextCursor`).

## 8. Exercise

**Beginner.** Write `services/posts.ts` (stub data model: posts table) with `listPosts`, `getPostBySlug`, and a `postToDto` mapper. No pages yet — just the service + its `Page<PostDTO>` contract. Typecheck-only: prove a client component *cannot* import it (add the `server-only` line).

**Intermediate.** Add `createPost`/`updatePost` with the full contract: `orgId` first param, `AppError` on conflict/404, DTO out. Write a **Vitest** test (module 20-02 skeleton) against an in-memory or ephemeral DB: create → list returns it; update with a foreign orgId → 404 (tenancy test *before authz exists* — the seam protects even from yourself).

**Production.** Refactor one real feature (or the capstone's product feature) so pages/actions contain *zero* Drizzle imports. Run `grep -rn "from '@/db" src/app src/features` — it must match only `services/` and `db/`. Commit with the grep output in the message.

## 9. Architecture Challenge

**Prompt:** The orders feature needs a "daily sales summary" for the dashboard: total revenue, order count, top product — computed from the `orders` + `order_items` tables for a given date range. Three engineers propose: (a) compute it live in a service on every dashboard load; (b) a nightly materialized table refreshed by a job; (c) a `use cache`'d service with `cacheLife('hours')`.

Choose per *role* (the user's own dashboard vs the admin's cross-org analytics vs the marketing landing page's "12,000 setups" stat). For each, state the staleness the choice implies and who invalidates what.

<details>
<summary>Model answer</summary>
User's own dashboard: (a) live service, but *scoped* (`eq(orgId, …)`) and cheap (one indexed aggregate per date range) — the user's numbers are decision-grade; staleness ≈ 0; no cache (or `seconds` life = dynamic hole). Caching a user-scoped aggregate is possible (keyed by orgId+range) but the invalidation story (after every order mutation → `updateTag('sales:'+orgId)`) is simpler to skip at this scale; measure first (module 18-01).
Admin cross-org: (b) materialized (or (a) with an index strategy — aggregates over all orgs are the expensive ones). The admin tolerates minutes-to-hours of staleness for a cross-org view; the job/`cacheLife('hours')` decides. If (c): `cacheTag('admin:sales')` revalidated by the job, not by order mutations (order-level tags for an all-orgs aggregate = invalidation storms — the "don't invalidate everything" rule, module 05-04).
Landing page stat: (c) `cacheLife('days')` — a marketing number, refreshed daily by the job or just left stale; nobody audits a landing-page stat to the minute. Invalidation: the job (or a deploy).
The lesson: the *same* data, three freshness contracts, three invalidation owners. Naming them is the design.
</details>

## 10. Official Documentation

- Fetching Data: https://nextjs.org/docs/app/getting-started/fetching-data
- Server and Client Components (boundary): https://nextjs.org/docs/app/getting-started/server-and-client-components
- Drizzle queries: https://orm.drizzle.team/docs/queries
- Data Security: https://nextjs.org/docs/app/guides/data-security
- `server-only` package: https://www.npmjs.com/package/server-only

## 11. What You Should Know Before Continuing

- [ ] I can state the 5 contract rules of the service layer and where each one shows up in the code
- [ ] My services: orgId-first, DTO-out, AppError-thrown, reads-cached, writes-direct
- [ ] `import 'server-only'` is in `db/` and I've watched a client import fail the build
- [ ] I know why caching belongs on *stable service functions*, not page renders
- [ ] I've grepped my app: zero ORM imports outside `services/` and `db/`

**Next:** Module 18 — Parallel Fetching & Waterfalls (the page's data topology).
