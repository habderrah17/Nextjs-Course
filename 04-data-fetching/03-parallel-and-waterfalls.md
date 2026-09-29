# Module 18 — Parallel Fetching & Killing Waterfalls

**Phase 4: Data Fetching · Module 18 of 101**

> **Where does this run?** `[SERVER]` (and the same topology rules apply to client islands). The rule: **independent reads run concurrently; dependent reads chain — and you can see the difference in the waterfall.**

---

## 1. Concept — Waterfalls are how pages get slow

A waterfall is a chain of `await`s where each step *could* have started earlier:

```
BAD (sequential, ~1.2s of pure waiting):
await cookies()          0.05s
await getSession()       0.2s   (DB lookup)
await getOrg()           0.2s
await listProducts()     0.4s
await getRevenueSummary() 0.35s   ← could have run DURING listProducts
await getRecentActivity() 0.15s   ← ditto
────────────────────────────────
total ≈ 1.35s before the first section can render
```

The fix is almost never "faster queries" — it's **topology**: which reads are independent (start together), which are dependent (must wait, and should wait for the *minimal* prerequisite), and which can be *deferred* (streamed behind a Suspense fallback while the rest renders).

## 2. Mental Model — the three topologies

```
1. PARALLEL (independent):        2. CHAINED (dependent):          3. DEFERRED (not needed yet):
   ┌─ A ───────┐                     A ──▶ B ──▶ C                    shell ─▶ <Suspense>
   ├─ B ──┐     │  (B needs A's        (C needs B's result)            └─ slow section streams in
   ├─ C ─┤     │  result — a real                              while the shell paints
   └─────┴─────┘  dependency, not a habit)
```

**The discipline:** before writing a second `await` in a render, answer: *does this read's input come from the previous read?* If no → `Promise.all`. If yes → chain (and name the dependency in a comment). If it's "not needed for first paint" → Suspense hole.

`Promise.all` vs `Promise.allSettled`: `all` fails fast (one rejected read fails the whole page → error boundary); `allSettled` lets you render *with* the failure (a section degrades, the page survives). Marketing data = `allSettled` (a dead external API shouldn't 500 the landing page); auth+core-data = `all` (fail loud).

## 3. Production Code

### 3.1 The dashboard, correct topology

`FILE: src/app/(app)/dashboard/page.tsx` (production pattern — [SERVER])

```tsx
import { headers } from 'next/headers'
import { Suspense } from 'react'
import { auth } from '@/lib/auth'
import { listOrders } from '@/services/orders'
import { getRevenueSummary } from '@/services/analytics'
import { getRecentActivity } from '@/services/activity'
import { DashboardSkeleton } from '@/components/dashboard-skeleton'

export default async function DashboardPage() {
  // CHAINED (real dependency): everything user-specific needs the session first.
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) throw new Error('unreachable: layout gated this route')
  const orgId = session.organizationId

  // PARALLEL from here: the three sections are independent of each other.
  // They also run INDEPENDENTLY of each other's completion because each is an
  // async child inside its own <Suspense> — the page resolves as each lands (module 06).
  return (
    <div className="grid gap-6">
      <Suspense fallback={<DashboardSkeleton section="revenue" />}>
        <RevenueSummary orgId={orgId} />
      </Suspense>
      <Suspense fallback={<DashboardSkeleton section="orders" />}>
        <OrdersSection orgId={orgId} />
      </Suspense>
      <Suspense fallback={<DashboardSkeleton section="activity" />}>
        <RecentActivity orgId={orgId} />
      </Suspense>
    </div>
  )
}

// Each section is an async component — its await starts the moment the render
// reaches it, and the <Suspense> above lets the OTHER sections stream out first.
async function RevenueSummary({ orgId }: { orgId: string }) {
  const summary = await getRevenueSummary(orgId)     // cached 'hours' (module 05-06)
  return <RevenueCard summary={summary} />
}
```

Why this is *both* parallel *and* streamed: React renders the page, hits each async child, and each child's promise starts immediately — no section waits for another; whichever resolves first emits HTML first. There is no `Promise.all` needed *at the page level* because the Suspense topology *is* the parallelism control. (Use `Promise.all` *inside* a section that needs multiple reads — next.)

### 3.2 Inside a section: explicit parallelism

`FILE: src/app/(app)/orders/[id]/page.tsx` (production pattern — [SERVER])

```tsx
export default async function OrderPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params

  // PARALLEL: order, its items, and the shipping status are three independent reads
  // (items need the order *id* — which we already have; not the order row).
  const [order, items, shipping] = await Promise.all([
    getOrderById(id),                       // throws AppError(404) if missing → notFound() in caller
    listOrderItems(id),
    getShippingStatus(id),                  // external API, 'minutes'-cached (module 16)
  ])
  if (!order) notFound()

  return (
    <div className="grid gap-6 lg:grid-cols-[2fr_1fr]">
      <OrderDetails order={order} items={items} />
      {/* Shipping streams separately if you want the page to paint before it resolves: */}
      <Suspense fallback={<ShippingSkeleton />}>
        <ShippingPanel status={shipping} />
      </Suspense>
    </div>
  )
}
```

### 3.3 `allSettled` for the landing page (degrade, don't die)

`FILE: src/app/(marketing)/page.tsx` (excerpt — [SERVER])

```tsx
export default async function HomePage() {
  // Marketing page: a dead external dependency must NOT 500 the landing page.
  const [stats, testimonials, integrations] = await Promise.allSettled([
    getHeroStats(),                          // internal, fast
    getFeaturedTestimonials(),                // internal, cached 'days'
    fetchFeaturedIntegrations(),              // external, 'minutes'-cached
  ])
  return (
    <HomeSections
      stats={stats.status === 'fulfilled' ? stats.value : FALLBACK_STATS}
      testimonials={testimonials.status === 'fulfilled' ? testimonials.value : []}
      integrations={integrations.status === 'fulfilled' ? integrations.value : []}
    />
  )
  // A missing section renders an empty/neutral state — logged server-side (module 21-01),
  // never a raw 500 on the most important URL in the business.
}
```

### 3.4 The client side (islands)

The same topology rules apply to client fetches (rare — module 13): `useEffect` chains that `await` sequentially when the reads are independent are the SPA habit re-appearing. If a client island *must* fetch two independent things (the on-demand feature data of module 14's challenge), `await Promise.all([fetchA, fetchB])` — and prefer lifting both to the server parent as props (the default).

## 4. How to find a waterfall (the operational skill)

1. **Dev server logs**: Next 16's `Render` timing per route — a 2s render on a 3-query page is a waterfall (or a slow query; module 18-04 distinguishes).
2. **Waterfall view**: DevTools Network (client) / a DB statement log or `pg_stat_statements` (server) — if queries A→B→C start *after* each other ends and don't need each other's results, they're a waterfall.
3. **The grep test**: `grep -n "await" src/app/.../page.tsx` — more than two sequential `await`s in a page body *before any render* is the code smell; each one must name its dependency.
4. **The cache check**: with Cache Components, a "waterfall" of *cached* reads is nearly free (cache hits are in-memory) — profile *uncached* work first.

## 5. Common Mistakes

| Mistake | Fix |
|---|---|
| `Promise.all`ing reads that have a *hidden* dependency (B needs A's id) | Runtime error or wrong data; chain explicitly and comment the dependency |
| `Promise.all` where one read is a 5s external API — everything waits for it | Suspense hole for the slow one; the fast ones render first |
| Sequential `useEffect` fetches in an island | Lift to the server parent (default) or `Promise.all` |
| Caching the *whole page* to hide a waterfall | You've frozen the data (stale) and masked the topology (next cold miss is brutal) | Fix the topology; cache per-read with honest lives |
| Forgetting the session read is a dependency root | Everything user-specific chains on it — do it once, at the top, not per-section |
| `allSettled` everywhere "to be safe" | Swallows real failures (a broken core query degrades silently) | `all` for core paths; `allSettled` for optional/external enrichment |

## 6. Security Notes

- Parallel reads don't change the threat model — but they *do* multiply it: two external fetches = two rate-limit/SSRF/validation surfaces (module 19-02). Each external read keeps its own validation (module 16).
- The session read at the top is *the* auth check for the page; parallel sections must **not** each re-derive authorization differently (same `session` object flows down — single source of truth).

## 7. Performance Notes

- Waterfall elimination is usually the *biggest* TTFB win available (bigger than any single query index) — measure before/after with the dev Render logs.
- Suspense parallelism + cache = the "shell paints in 100ms, data lands as it does" experience the course keeps promising; the dashboard lab (module 28) makes it visible.
- N+1 *is* a topology bug: N reads where 1 batched read suffices (module 18-04, with the fix).

## 8. Exercise

**Beginner.** In a scratch server page, write three reads: session (50ms fake delay), products (400ms), revenue (300ms). Version 1: sequential `await`s — measure with `console.time`. Version 2: session first, then the two in `Promise.all`. Version 3: each in its own Suspense hole. Log all three timings and the *per-section* paint times (dev server).

**Intermediate.** Add a 5s fake "external integrations" read to the marketing page. With `Promise.all` (v1) the whole page waits 5s; with an `allSettled` + Suspense split (v2) the hero paints in ~0.5s. Screenshot both waterfalls. Document which pattern you'd use for (a) a checkout page's address validation, (b) a blog page's related-posts.

**Production.** Instrument the capstone dashboard's *real* queries (add a temporary `console.time` in each service, or enable Drizzle query logging). Find at least one real waterfall or N+1, fix it (topology or batching), and record the Render-time delta. This is the first "performance investigation" of the course — module 18-01 formalizes the method.

## 9. Architecture Challenge

**Prompt:** The checkout page needs: cart (your DB), shipping options (external carrier API, 800ms, sometimes 4s), payment methods (your DB), fraud check (external, 300ms, must complete *before* payment is submitted but not before the page paints).

Design: the read topology (parallel/chained/deferred), where the fraud check runs (it's a *mutation-adjacent* call — server, but when?), the failure behavior for the carrier API being slow *while the user is filling the form*, and the one place a `Promise.all` would be *wrong*.

<details>
<summary>Model answer</summary>
Reads on page load: cart + payment methods in `Promise.all` (both yours, both needed to render the form; failure = 500/error boundary — `all`, core path). Shipping options: **Suspense hole** with a "calculating shipping…" skeleton — the 800ms–4s carrier latency must not delay first paint; `allSettled`-style degradation (show "shipping calculated at next step" if it fails).
Fraud check: NOT a page read. It runs inside the `placeOrder` **Server Function** (module 07) — it guards the *mutation*, not the render; running it at page load is both wasteful (the user may leave) and wrong (fraud checks are point-in-time; the order data is only complete at submit). Sequence inside the action: validate → fraud check → charge → create order, each with its failure branch (decline = user-facing 422, fraud timeout = fail closed with a clear message).
Carrier slow mid-form: the Suspense hole already resolved or is still pending — either way the form is usable; if the user submits before options arrive, the action *re-fetches* shipping server-side (the authoritative price is computed at submit, not rendered — the render is a preview).
`Promise.all` would be *wrong* around the carrier read: it would hold the entire checkout's first paint hostage to an external 4s tail — the anti-pattern in one sentence.
</details>

## 10. Official Documentation

- Fetching Data (parallel, `Promise.all`): https://nextjs.org/docs/app/getting-started/fetching-data
- Suspense & streaming: https://nextjs.org/docs/app/getting-started/fetching-data#streaming
- How Revalidation Works (timing internals): https://nextjs.org/docs/app/guides/how-revalidation-works
- React — Suspense: https://react.dev/reference/react/Suspense

## 11. What You Should Know Before Continuing

- [ ] I can classify any two reads as parallel/chained/deferred and write the matching code
- [ ] I know when `all` vs `allSettled` vs Suspense-hole is the right tool (and the failure semantics of each)
- [ ] I can find a waterfall from dev logs + the grep test
- [ ] I know the fraud-check placement principle (mutations guard mutations, not renders)
- [ ] I've measured a real before/after Render-time improvement

**Next:** Module 19 — The Request Lifecycle (tracing one request end-to-end; closes Phase 4).
