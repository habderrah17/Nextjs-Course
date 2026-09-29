# Module 19 — The Request Lifecycle: One Request, End to End

**Phase 4: Data Fetching · Module 19 of 101 (closes Phase 4)**

> **Where does this run?** Everywhere, in order. This module is the *exam* for Phases 1–4: if you can narrate this lifecycle for any URL in the capstone, you have the framework's execution model.

---

## 1. Concept — The lifecycle as a numbered protocol

Take `GET /orders/8f3a` (a logged-in user opening an order detail) through the capstone. Every step names **where** it runs and **what** can go wrong:

```mermaid
sequenceDiagram
    participant B as Browser
    participant C as CDN / Edge
    participant P as proxy.ts [SERVER]
    participant R as Router + layouts [SERVER]
    participant L as (app) layout [SERVER]
    participant PG as order page [SERVER]
    participant SVC as services [SERVER]
    participant DB as Postgres
    participant CL as Client islands [CLIENT]

    B->>C: GET /orders/8f3a (cookie: session_token)
    C->>P: (cache miss for this dynamic URL)
    P->>P: matcher hits /orders/* → has cookie? → continue (optimistic only)
    P->>R: route match: (app)/orders/[id]
    R->>L: render root layout → (app) layout
    L->>SVC: getSession(headers) → session row
    SVC->>DB: SELECT session…
    DB-->>L: session (user, orgId)
    L->>PG: layout OK → render page (orgId + id flow down)
    PG->>SVC: getOrderById(id) + listOrderItems(id) [Promise.all]
    SVC->>DB: SELECT … WHERE id=… AND orgId=…   (tenancy in the WHERE)
    DB-->>PG: rows → DTOs
    PG-->>B: HTML shell (incl. Suspense fallbacks) — TTFB moment
    PG-->>B: streamed: order sections resolve
    B->>B: paint; hydrate islands (OrderTimeline, CancelButton)
    Note over B,CL: user clicks "Cancel"
    CL->>P: POST (Server Function: cancelOrder)
    P->>SVC: validate → cancel → updateTag('orders')
    SVC-->>CL: RSC re-render (fresh row) — no full reload
```

**The four execution times, mapped:**
1. **Build**: the *layouts* + any `use cache`'d reads with long-enough lives are prerendered (static shell); `generateStaticParams` values pre-rendered; fonts/compiled.
2. **Request**: proxy → layout session read → page reads → stream. (Everything in the sequence diagram above.)
3. **Hydration**: client islands initialize; effects run; the interactive layer exists.
4. **Navigation/Action**: subsequent clicks = RSC delta (no document) or POST → re-render.

## 2. Mental Model — the three "clocks" of a request

When you debug "why is this slow/stale/wrong," you're looking at one of three clocks:

| Clock | What it measures | Tools |
|---|---|---|
| **The render clock** | server: build→request→stream (TTFB, section latency) | dev `Render` logs, Web Vitals (TTFB/LCP) |
| **The hydration clock** | client: HTML→interactive (INP precursor) | dev overlay, Lighthouse "Interactive" |
| **The freshness clock** | cache: when did this data last regenerate? who invalidated it? | module 05's five questions; dev overlay cache insights |

A "slow page" is one clock; a "wrong page" is usually the freshness clock (stale cache) or the tenancy check (wrong org scope). **Naming the clock before fixing it** is the debugging discipline that runs through module 25.

## 3. Production Code — tracing aids (the instrumentation you keep)

### 3.1 The dev log you already have

`next dev` prints, per request:

```
Compile /orders/8f3a in 45ms (next dev)        ← routing + compile (Turbopack)
Render  /orders/8f3a in 182ms                  ← YOUR code: session read + queries
```

A 180ms `Render` with two cached reads is suspicious (cache misses?); with two cold external calls it's fine. **Read these numbers every dev session** — they are the cheapest performance instrument in the stack.

### 3.2 A request-scoped trace (for the debug lab, module 25)

`FILE: src/instrumentation.ts` (simplified example — [SERVER])

```ts
// Next.js calls register() on server startup (Node runtime).
export async function register() {
  if (process.env.NEXT_RUNTIME === 'nodejs') {
    await import('./src/lib/tracer')
  }
}
```

`FILE: src/lib/tracer.ts` (production pattern — [SERVER])

```ts
import { AsyncLocalStorage } from 'node:async_hooks'

// A per-request context: an ID that flows into every log line this request produces
// (module 21-01). This is the skeleton of the observability layer — deliberately small.
const storage = new AsyncLocalStorage<{ requestId: string }>()

export function runWithRequest<T>(id: string, fn: () => T): T {
  return storage.run({ requestId: id }, fn)
}

export function currentRequestId(): string {
  return storage.getStore()?.requestId ?? 'no-request'
}
```

(You'll wire this into the proxy → `runWithRequest(randomUUID(), …)` pattern in module 21-01; here it just makes the concept concrete: *one request = one ID = one correlatable log stream*.)

## 4. The failure map (what breaks at each step)

| Step | Failure | Symptom | Module |
|---|---|---|---|
| proxy | matcher too greedy / slow logic | *all* matched pages slow; wrong rewrites | 05-09, 11 |
| layout session read | session table down | authed pages 500 (error boundary); marketing unaffected (no session read) | 10-05 |
| page read (tenancy) | missing `orgId` in WHERE | **cross-tenant data render** — the security bug | 11-02, 19 |
| external read | provider down | Suspense section shows error fallback (if designed) or page 500 | 16 |
| stream | one section throws mid-stream | that section's `error.tsx`; rest of page intact (if boundaries exist) | 08, 06 |
| hydration | client first render ≠ server HTML | dev overlay diff; prod: patched DOM, console noise | 13, 25-01 |
| action POST | missing validation | garbage in DB / 500 | 07-03 |
| action revalidation | wrong tag (or none) | user sees stale data after their own mutation | 05-04, 25-01 |

**The exam question for the rest of the course**: for any bug you encounter, place it on this map *before* reading code.

## 5. Common Mistakes

| Mistake | The actual cause |
|---|---|
| "The page is slow" with no clock named | Measure first (module 18-01): which of the three clocks? TTFB 3s? or hydration 3s? or data 10 minutes stale? |
| Debugging a cache bug by clearing browser cache | The browser cache is layer 1 of 6 (module 05-01); the bug is usually layer 3 (Next data cache) or the freshness clock |
| Assuming the proxy runs for `/api/*` (or doesn't) | The `matcher` decides; read it | 05-09 |
| Believing "the server rendered it, so the client re-render is cheap" | Hydration cost is proportional to client subtree size | 13, 18-03 |
| Adding logs by `console.log`ing in the page render and not noticing the request-ID-less soup | Structured, request-scoped logging (module 21-01) | — |

## 6. Security Notes

- The lifecycle *is* the security review: proxy (not a boundary) → layout (first enforcement) → page read (tenancy) → action (validation + authz) → revalidation (staleness is a *data* security property: a stale "suspended user" row that renders UI is a live authorization bypass — module 11-03).
- Every step that touches the session must use the *same* session resolution (one function, `lib/auth.ts`) — two resolvers drift (one checks `revokedAt`, the other doesn't).

## 7. Performance Notes

- TTFB = proxy + layout + first page read. The layout's session read is on *every* authed TTFB — keep it a single indexed lookup (module 10-05 shows the Cache Components interaction).
- The stream is your friend: TTFB should reflect the *shell*, not the slowest section. If TTFB includes a 4s carrier call, your Suspense topology is wrong (module 18).
- Hydration is budget: the islands list from Architecture Review #1 is the hydration cost.

## 8. Exercise

**Beginner.** Narrate the lifecycle of `/products` (public) out loud, step by step, naming the runtime at each step. Then of `/dashboard`. Write both narrations in `docs/lifecycle.md`. Where do they first diverge? (Answer: the `(app)` layout's session read — everything after is downstream of that branch.)

**Intermediate.** Add the §3.2 tracer. In the proxy, assign a `requestId` (UUID) per request. In two services, log `currentRequestId()` + query name + duration. Generate two overlapping requests (two browser tabs) and confirm the log lines are correlatable per-request. This is module 21-01's foundation, built early on purpose.

**Production.** Execute the failure map *for real*: for each row, deliberately cause the failure in dev (kill Postgres; break the session cookie; remove an `orgId` WHERE; add a 4s fake external read; throw in one Suspense section; force a hydration mismatch). For each: record the exact user-visible symptom, the dev-overlay/console output, and the *clock* it belongs to. This table is your debugging field guide — and half of module 25's lab.

## 9. Architecture Challenge

**Prompt:** A support ticket: "My customer's order page shows the old status ('pending') but the admin console (opened by a staff member 20 seconds later) shows 'paid'. The payment webhook succeeded." No code changes since last week.

Walk the lifecycle: which step *should* have made both views consistent? What is the most likely defect (name the mechanism, not just "caching"), what is the *safe* temporary mitigation (don't say "clear all caches"), and what is the permanent fix including the invalidation design (tag names, who calls it, which semantics — `updateTag` vs `revalidateTag`)?

<details>
<summary>Model answer</summary>
Walk: the webhook (Route Handler, module 08) marked the order paid in the DB — step "SVC → DB" succeeded. The *user's* order page read `getOrderById` — if that read is `use cache`'d (say, `cacheLife('hours')` for "order details don't change much"), the user's view is a *cached* pending from before the webhook. The admin console's read is either uncached (dynamic) or on a different tag/profile — so it sees fresh 'paid'. The defect: **the webhook mutated the DB but did not invalidate the order-detail cache tag** — the freshness clock and the DB are out of sync. (This is the "mutation without revalidation" bug — the most common production Next.js bug, module 25-01.)
Mitigation (safe, no global flush): make the *user-facing* order-status read short-lived (`cacheLife('seconds')` → dynamic hole, always fresh for status) while keeping the *order details* (items, addresses) on the longer life — status is the volatile field; split the read. Alternatively revalidate the specific tag from the webhook immediately.
Permanent fix: the webhook handler, after `markOrderPaid(orderId)`, calls `revalidateTag('order:' + orderId, 'max')` (stale-while-revalidate is correct here: other users' views refresh in the background; the *paying user* hits the `seconds`-life status read anyway). Tag design: `order:{id}` (fine-grained, per-order) + `orders` (list views) — the webhook invalidates both. The admin's console stays uncached for status (it's an ops view; freshness is its product). Document all three reads' profiles in the caching case study (module 05-06) — that table is the prevention.
</details>

## 10. Official Documentation

- The App Router (architecture): https://nextjs.org/docs/app/getting-started
- Request APIs (async): https://nextjs.org/docs/app/getting-started/server-and-client-components
- Instrumentation: https://nextjs.org/docs/app/api-reference/other-apis/instrumentation
- How Revalidation Works: https://nextjs.org/docs/app/guides/how-revalidation-works
- Building (what happens at build): https://nextjs.org/docs/app/guides/building

## 11. What You Should Know Before Continuing

- [ ] I can narrate the full lifecycle of any capstone URL, naming the runtime at each step
- [ ] I know the three clocks (render/hydration/freshness) and can name the clock of a symptom
- [ ] I have the failure map and have caused at least 5 of its rows on purpose
- [ ] I have request-scoped logging working (module 21's foundation)
- [ ] I can diagnose the "stale status" ticket: mechanism, mitigation, permanent invalidation design

**Phase 4 complete.** The capstone's data path is real: Postgres + Drizzle + services + DTOs + parallel topology. **Next:** Phase 5 — Caching, deeply. Module 20: the six cache layers and the five questions (the mental model the whole course keeps deferring).
