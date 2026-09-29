# Module 25 — Case Study: Caching the Capstone Dashboard

**Phase 5: Caching · Module 25 of 101**

> **Where does this run?** `[SERVER]` (the dashboard is the course's central server-rendered surface) with `[CLIENT]` islands. This module is the **worked example** of modules 20–24: every section of the real dashboard, answered against the five questions, with the reasoning *written out* — the format a senior reviewer would demand.

---

## 1. Concept — The dashboard is the hardest caching surface you'll build

A dashboard concentrates every caching difficulty in one page: **user-specific** data (never public), **mixed change rates** (revenue: slow; notifications: instant), **mixed actors** (the user sees their own org; an admin sees all), **mutations that must reflect immediately** for the actor, and a **live-ish** feel users expect. The old model's answer was "render the whole thing per request, every time" — correct but slow. The Cache Components answer is a **per-section contract** — which is exactly what this module writes.

**The dashboard's sections** (the capstone's, from the PRD — module 24-01):

```
/app/(app)/dashboard
├── RevenueSummary        (revenue, order count, top product — last 30d, this org)
├── RecentOrders          (the org's last 10 orders, live-ish statuses)
├── RecentActivity        (org event feed: product published, member joined, …)
├── Analytics             (traffic/conversion charts — this org, last 30d)
└── [client islands]      (filters, cancel buttons, the "last updated" chip)
```

## 2. The Caching Document (the artifact — every row reasoned)

`FILE: docs/caching-inventory.md` — the dashboard section:

| Surface (service) | WHAT | WHERE | HOW LONG (profile — why) | WHO invalidates (tag ← mutation) | USER SEES after invalidation |
|---|---|---|---|---|---|
| `getRevenueSummaryCached(orgId)` | data (aggregate) | L3 memory (per-instance; no remote — see note) | **`hours`** — decision-grade, not real-time; a 1h-old revenue number changes no user decision; the org's order volume makes `minutes` a needless regeneration load | `'revenue:{orgId}'` ← order paid/shipped/cancelled (webhook + actions), refund actions | **Actor**: immediate (`updateTag` in the action that changed it). **Others in the org**: SWR — stale until their next visit's background regen (≤1h, usually seconds-minutes of traffic) |
| `getRecentOrdersCached(orgId)` | data (list) | L3 | **`minutes`** — glanced constantly; order *status* is the volatile field (module 19's ticket); a 5-min-stale status on a list is tolerable, the detail page is fresher | `'orders'` + `'order:{id}'` ← order mutations (actions + payment webhook) | **Actor**: immediate. **Fleet**: SWR (the `minutes` life caps the staleness anyway — double coverage, deliberate) |
| `getOrderCached(id, orgId)` (detail, linked from the list) | data | L3 | **`minutes`** (the detail is where staleness *hurts* — module 19's ticket fix: status fields live; the *detail* is `minutes`, not `hours` — the user clicks a row and expects the current state) | same tags as the list + the specific `'order:{id}'` | Actor immediate (updateTag on their own mutation); fleet SWR |
| `getRecentActivityCached(orgId)` | data (feed) | L3 | **`minutes`** — a feed whose newest item is 5 minutes old is "live enough"; the feed is *append-only* (new events add; old ones don't change) → revalidation is cheap (append semantics) | `'activity:{orgId}'` ← the org-event emitters (publish, member join, order events — the *same* mutations that touch orders/activity tags) | Actor immediate (their own event is the newest row — the action appends then invalidates); fleet SWR |
| `getAnalyticsCached(orgId, range)` | data (aggregate) | L3 | **`hours`** — analytics is a *report*, not a feed; the range is in the args (the key includes `range` — switching date ranges is a *different entry*, module 21's key contract) | `'analytics:{orgId}'` ← the nightly rollup job (via a service call / admin route) + order mutations (light touch: `revalidateTag` only — the nightly job is the *primary* refresher) | **No actor story** (users don't mutate analytics directly) — fleet SWR; the nightly job makes it *truly* fresh once a day |
| session read (`getSession` in the layout) | request data | **NOT CACHED** (L4 at best) | n/a — auth-relevant: a suspended user's stale "active" cache is a **bypass** (module 20/23 security line) | n/a | always fresh (per request) |
| the org's `orgId` resolution | request data | NOT CACHED | n/a — it *is* the tenancy key; caching it would freeze membership changes | n/a | always fresh |

**The notes that make the document honest:**

- **Why no `remote` cache here?** The dashboard's entries are **per-org** (high cardinality × per-instance memory). On Docker (one long-lived process), in-memory is a *shared* cache — fine. On serverless (many instances), each instance has its own memory → the *first* request per instance per org pays the cold query. The decision: at the capstone's scale (one org per customer, moderate traffic), per-instance cold hits are rare (a customer's requests hit warm instances) — measure before adding Redis (`'use cache: remote'` or a `cacheHandlers` Redis). The document *names* the decision and the escape hatch instead of adding infra by default (module 22's "default in-memory" rule).
- **Why is the actor's freshness `updateTag` and not a longer life?** Because "I did a thing and the dashboard doesn't reflect it" is the #1 support ticket in any SaaS (module 19's ticket, generalized). The actor guarantee is *cheap* (one tag invalidation in the action) and *non-negotiable*; the fleet gets SWR because *their* staleness is bounded by the life anyway.
- **Why does the order detail differ from the list?** The list is *scanning* (5-min-old statuses are fine); the detail is *decisioning* (the user clicked a row — they expect current state). Same data, two surfaces, two contracts — the per-*surface* (not per-*dataset*) decision, module 23's product-price generalization.

## 3. Architecture — the dashboard's request, annotated

```mermaid
sequenceDiagram
    participant U as User's browser
    participant CDN as CDN (L2)
    participant S as Next server (L3 in-memory)
    participant DB as Postgres (L6)

    U->>CDN: GET /dashboard (session cookie)
    Note over CDN: the (app) layout is SESSION-SCOPED → not globally cacheable;<br/>the *public* parts of the shell could be, but the shell here is per-session
    CDN->>S: (authed route: request-time)
    S->>S: proxy: has cookie? continue (optimistic)
    S->>DB: session row (L4/request-scoped — the first real read, ~5ms indexed)
    S->>DB: org resolution (membership row, ~2ms)
    par four independent sections (module 18: parallel via Suspense holes)
        S->>S: getRevenueSummaryCached(orgId) — L3 hit? (common: yes, ~1ms)
        S->>S: getRecentOrdersCached(orgId) — L3 hit?
        S->>S: getRecentActivityCached(orgId) — L3 hit?
        S->>S: getAnalyticsCached(orgId, range) — L3 hit?
    end
    Note over S: cold entries (post-deploy / new org): the queries run (~20–80ms each,<br/>parallel), stream in as they land — skeletons for the slow ones
    S-->>U: HTML (shell: layout chrome + section skeletons) — TTFB ≈ session+org+fastest section
    S-->>U: stream: revenue → orders → activity → analytics (land order)
    U->>U: hydrate islands (filters, cancel buttons)

    Note over U,DB: later: user cancels an order
    U->>S: POST cancelOrder (Server Function)
    S->>DB: cancel (committed)
    S->>S: updateTag('orders') + updateTag('order:{id}') + revalidateTag('orders','max') + …
    S-->>U: RSC re-render — the dashboard re-fetches its sections;<br/>'orders' is FRESH (updateTag), the rest serve from cache (still valid)
```

**Read the diagram as the model:** TTFB is the *layout + session + org* path (the per-session shell's cost — small, two indexed lookups); the sections stream independently (module 18's topology *is* the caching story — the holes are the per-section entries); the mutation's re-render revalidates *only* the tags it touched (module 23's spec) — the revenue section doesn't regenerate because an order was cancelled *unless* the cancellation also changed revenue (it does — so the action invalidates `'revenue:{orgId}'` too; the spec is **complete**: every tag whose truth changed).

## 4. Production Code — the dashboard page, final form

`FILE: src/app/(app)/dashboard/page.tsx` (production pattern — [SERVER]; the complete file)

```tsx
import { headers } from 'next/headers'
import { Suspense } from 'react'
import { redirect } from 'next/navigation'
import { auth } from '@/lib/auth'
import { getActiveOrgId } from '@/services/organizations'
import { getRevenueSummaryCached } from '@/services/analytics'
import { getRecentOrdersCached } from '@/services/orders'
import { getRecentActivityCached } from '@/services/activity'
import { getAnalyticsCached } from '@/services/analytics'
import { DashboardSkeleton } from '@/components/dashboard-skeleton'
import { RevenueCard } from '@/features/analytics/components/revenue-card'
import { OrdersTable } from '@/features/order/components/orders-table'        // [CLIENT] island
import { ActivityFeed } from '@/features/activity/components/activity-feed'
import { AnalyticsCharts } from '@/features/analytics/components/analytics-charts' // [CLIENT] island (charts)

export default async function DashboardPage() {
  // The session/org reads are REQUEST-SCOPED (the caching document's NOT-CACHED rows).
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) redirect('/login')
  const orgId = await getActiveOrgId(session.user.id)
  if (!orgId) redirect('/onboarding/select-org')

  return (
    <div className="grid gap-6">
      <Suspense fallback={<DashboardSkeleton section="revenue" />}>
        <RevenueSummary orgId={orgId} />
      </Suspense>
      <Suspense fallback={<DashboardSkeleton section="orders" />}>
        <RecentOrders orgId={orgId} />
      </Suspense>
      <div className="grid gap-6 lg:grid-cols-2">
        <Suspense fallback={<DashboardSkeleton section="activity" />}>
          <RecentActivity orgId={orgId} />
        </Suspense>
        <Suspense fallback={<DashboardSkeleton section="analytics" />}>
          <Analytics orgId={orgId} />
        </Suspense>
      </div>
    </div>
  )
}

// — sections: each an async child; cached reads are the services' 'use cache' entries (module 21)

async function RevenueSummary({ orgId }: { orgId: string }) {
  const summary = await getRevenueSummaryCached(orgId)   // hours · 'revenue:{orgId}'
  return <RevenueCard summary={summary} />
}

async function RecentOrders({ orgId }: { orgId: string }) {
  const { items, nextCursor } = await getRecentOrdersCached(orgId)   // minutes · 'orders'
  return (
    <section aria-label="Recent orders" className="space-y-2">
      <OrdersTable orders={items} nextCursor={nextCursor} />   // island: cancel button → cancelOrder action
    </section>
  )
}

async function RecentActivity({ orgId }: { orgId: string }) {
  const events = await getRecentActivityCached(orgId)          // minutes · 'activity:{orgId}'
  return <ActivityFeed events={events} />
}

async function Analytics({ orgId }: { orgId: string }) {
  const data = await getAnalyticsCached(orgId, '30d')          // hours · 'analytics:{orgId}'
  return <AnalyticsCharts data={data} />                        // island: the chart library is client-only
}
```

**The islands' mutation wiring** (module 07's pattern, shown once for the dashboard): `OrdersTable`'s cancel button calls `cancelOrder(id)` (the action with the §3 of module-23 invalidation spec) → the action `updateTag`s → the RSC re-render streams a fresh orders section → the button's row updates, *without the client refetching anything*. The client never "invalidates" — it *receives*.

## 5. Common Mistakes (the dashboard's specific failures)

| Mistake | The symptom | The document's answer |
|---|---|---|
| Caching the whole dashboard as ONE UI-level entry | Every section re-renders on any mutation; one slow section blocks all | Per-section data-level entries (the document's rows) — invalidation is per-tag, rendering is per-section |
| The session read inside a `'use cache'` scope | Build error — or, worse in legacy code, a *public* cache of a user's session | Request-scoped (the NOT-CACHED row) |
| `'revenue:{orgId}'` but the invalidation uses `'revenue'` | The tag never matches (the typo-tag bug, module 23) | `TAGS.revenue(orgId)` from the typed module; the audit (module 23's exercise) catches the divergence |
| Analytics cached per `range` *and* the UI passes a fresh `Date` object as range | Key churn (the unstable-arg bug, module 21) | `range` is a **string** (`'30d'`) — the URL state (module 10), not a computed value |
| The actor's "cancel order" uses only `revalidateTag` (no `updateTag`) | "I cancelled, the list still shows pending for a minute" — the #1 ticket | `updateTag` for the actor (the document's USER-SEES column says so explicitly) |
| Adding a client `setInterval` refetch "for live feel" | A second data path, a second staleness story, a bundle cost | The `minutes`/`hours` lives *are* the freshness; a real live need is SSE (module 18's challenge) — not polling |

## 6. Security Notes

- **The per-tenancy key is the security of this page** (`orgId` in every cached entry's key): a missing key component = org A's revenue rendered for org B. The document's rows *name* the key's args — that naming is the control. (Module 11-02's cross-tenant test targets exactly these entries.)
- **The NOT-CACHED rows are the authz rows**: session + org resolution per request means a *suspended* member's dashboard 401s on the next request (module 23's suspend challenge, applied) — no cached "you're in this org" survives a removal.
- The admin dashboard (all orgs) is a *different* surface with a *coarser* key (no orgId — it's the admin's aggregation) and a *stricter* gate (the `(admin)` layout's role check, module 06): the caching document gets a second, admin-scoped section in module 17-02.

## 7. Performance Notes

- **The TTFB budget** (the number to hit): session (5ms) + org (2ms) + fastest section (a cache hit, 1ms) → **TTFB ≈ 10–50ms + network** — the *shell* (layout chrome + skeletons) is effectively instant; the sections land 20–300ms later, independently. Measure it (module 18-01): TTFB should be *network-bound*, not query-bound, in the warm case.
- **The cold case** (post-deploy, new org): all four sections cold → four parallel queries (~50–200ms) → the page is still *fast* (parallel, streamed) but *not instant* — the skeletons make it *perceived* as fast. This is the honest performance claim: **warm = instant, cold = graceful** — the cache architecture's actual contract.
- **The LCP element** is the RevenueCard (the dashboard's hero number) — it's in the `hours`-cached section: shell-eligible *for the data*, but the *page* is session-scoped (not globally CDN-cacheable) — so its "shell" is the *per-session* first render + the L3 entry. The LCP audit (module 24's exercise) records this nuance: the LCP is *data-cached*, not *page-cached* — different mechanism, same win (the query is a cache hit).

## 8. Exercise

**Beginner.** Build the dashboard page (§4) with stub services that implement the document's profiles (fake delays: revenue 400ms, orders 250ms, activity 150ms, analytics 600ms). Verify: (a) TTFB shows the shell first (dev logs); (b) sections land in *dependency order* (activity before revenue — the fast one wins); (c) a second visit (soft nav) is a cache hit (logs show no query executions). Screenshot the waterfall.

**Intermediate.** Implement the cancel flow end-to-end: `OrdersTable` → `cancelOrder` (module 23's action, full spec) → the RSC re-render. Then the **actor/fleet test**: two browser contexts (two "users", same org — use two incognito windows with the same seeded session, or a test that logs per-orgId entries): user A cancels; user A's view is immediate; user B's *next* view shows the cancellation after the SWR window (log the regeneration). Document the exact timing you observed.

**Production.** Run the **full invalidation audit** on the dashboard: for each of the ~8 mutations that touch dashboard data (order paid/shipped/cancelled, refund, product published, member joined, org renamed, analytics rollup), write the tag list it invalidates (from the actions/webhooks) and cross-check against the document's WHO column. Any mismatch = a bug (stale surface). Fix them; commit the corrected document. This is Architecture Review #2's core artifact — the review (module 23-04) signs it.

## 9. Architecture Challenge

**Prompt:** The dashboard gets a "compare with last period" toggle (a client island that switches `range` from `'30d'` to `'30d:compare'` — a URL param). The compare variant is a *different aggregate* (two periods, a delta). Options: (A) one cached function `getAnalyticsCached(orgId, mode)` where `mode ∈ {'30d', '30d:compare'}` (two entries per org); (B) two functions (single/compare) with separate tags; (C) cache the single-period data and *compute the delta client-side* from two period reads.

Choose, weighing: entry cardinality (2× per org), the compare read's *change rate* (same as single — data doesn't know about the UI toggle), the client-compute's correctness (the delta must match the server's — rounding, exclusions), and the URL-state interaction (module 10: the toggle is URL state — what does that imply about the *server* rendering of the compare view?). Defend with numbers where you can.

<details>
<summary>Model answer</summary>
(C) is the trap: computing the delta client-side from two *display-formatted* reads (rounded currency, formatted counts) produces deltas that don't match the server's authoritative compute (rounding error, excluded orders, currency) — a *correctness* failure, and it ships two data reads to the client for a computation the server should own. Rejected.
(B) is defensible but over-engineered *today*: two functions, two tags, two invalidation specs for a *view* of the same data — the compare is the *same* aggregates (this period + last period) with a delta; the invalidation is identical (the underlying order data). Two tags that must always be invalidated together = the "two owners, one truth" smell (module 23's rule 3) — a future refactor invalidates one and not the other.
(A) is right: the mode is a *view argument* — the key already includes all args (module 21), so `getAnalyticsCached(orgId, '30d:compare')` is a *separate entry* with the *same* life and *same* tag (`'analytics:{orgId}'` — the tag is on the *data*, not the view; invalidating the tag refreshes both views on next hit). Cardinality: 2 entries/org — trivial. Change rate: identical (the data's, not the UI's) — same profile. URL state: the toggle is `?mode=compare` (module 10) — the *server* renders the compare view when the param is present (the param flows into the `mode` arg — the page reads `searchParams` in the uncached scope, passes `mode` in — the module-24 pattern), so a shared link to `?mode=compare` renders correctly for any viewer (the toggle is *shareable* — the whole point of URL state). The invalidation spec is *unchanged* (the tag is data-scoped): the nightly job + order mutations refresh both entries.
The generalization (write it in the document): **view args (mode, range, format) join the key, not the tag; data args (orgId, the thing that changes) own the tag.** Views multiply entries cheaply; tags invalidate truth.
</details>

## 10. Official Documentation

- Caching: https://nextjs.org/docs/app/getting-started/caching
- Revalidating: https://nextjs.org/docs/app/getting-started/revalidating
- How Revalidation Works: https://nextjs.org/docs/app/guides/how-revalidation-works
- Fetching Data (streaming sections): https://nextjs.org/docs/app/getting-started/fetching-data

## 11. What You Should Know Before Continuing

- [ ] I can write a per-section caching document (the 5 columns) for any complex page — the dashboard is the template
- [ ] I know the actor/fleet distinction (`updateTag` vs `revalidateTag`) and why it's non-negotiable for the actor
- [ ] I can annotate a dashboard request (the sequence diagram) and name each layer's cost
- [ ] I know the cold-vs-warm contract (warm = instant, cold = graceful) and can measure both
- [ ] The invalidation audit (mutation → tags → document) is green for the dashboard
- [ ] I can state the view-args-join-the-key / data-args-own-the-tag generalization

**Next:** Module 26 — the Previous Model (the old fetch-cache world) and migrating a legacy codebase to Cache Components.
