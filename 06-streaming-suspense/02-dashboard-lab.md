# Module 28 — Lab: The Dashboard, Streaming, Measured

**Phase 6: Streaming & Suspense · Module 28 of 101 · the final module of Phase 6**

> **Where does this run?** The capstone dashboard (`[SERVER]` sections, `[CLIENT]` islands, `[BOTH]` boundaries) — run *here*, in dev, with the module-19 tracer. This is a **lab**: a fixed setup, numbered experiments, each with a *hypothesis* (predicted from module 27) and an *observed* result (the tracer log). The lab report is the phase-gate artifact.

---

## 1. Concept — Why a lab, not just a page

Module 25 *designed* the dashboard (the caching document); module 27 *explained* the streaming (the rules). This module **proves both, with numbers**, because the dashboard's "it feels instant" is a *claim* that should be *evidenced* before it's repeated in a standup. The lab's standing claim to verify:

> **On the capstone dashboard, the shell (layout + session + org) renders in < 100ms of server work; the four sections (Revenue, Orders, Activity, Analytics) resolve *independently* — in completion order — each behind a same-geometry skeleton; the warm case (entries hot) resolves all sections within ~50ms of the shell; the cold case (fresh build) resolves them within 150–400ms, *gracefully* (skeletons, no layout shift, per-section error isolation).**

Each experiment below tests one clause of that claim.

## 2. Setup — The lab bench

### 2.1 The instrumented services (fixed latencies, the lab's variables)

`FILE: src/services/stub.ts` (lab-only — [SERVER]; the lab uses *these* stubs, then re-runs with the real services at the end)

```ts
// The lab's controlled variables: fake latencies per section.
// The tracer (module 19) logs each entry's execution — that's the measurement.
const delay = (ms: number) => new Promise((r) => setTimeout(r, ms))

export const LATENCY = { revenue: 300, orders: 150, activity: 80, analytics: 500 } as const

export async function stubRevenue(orgId: string) {
  await delay(LATENCY.revenue)
  return { orgId, revenue: 48_210.55, orders: 312, topProduct: 'Aurora Stand' }
}
export async function stubOrders(orgId: string) {
  await delay(LATENCY.orders)
  return { items: [
    { id: 'o-1', number: '#1042', status: 'paid', total: 129.0, placedAt: '2026-09-26T10:12:00Z' },
    { id: 'o-2', number: '#1041', status: 'pending', total: 59.0, placedAt: '2026-09-26T08:41:00Z' },
    { id: 'o-3', number: '#1040', status: 'shipped', total: 219.5, placedAt: '2026-09-25T19:03:00Z' },
  ], nextCursor: null }
}
export async function stubActivity(orgId: string) {
  await delay(LATENCY.activity)
  return [
    { id: 'e-1', type: 'order.paid', label: 'Order #1042 was paid', at: '2026-09-26T10:12:00Z' },
    { id: 'e-2', type: 'member.joined', label: 'Dana joined the team', at: '2026-09-25T09:00:00Z' },
  ]
}
export async function stubAnalytics(orgId: string) {
  await delay(LATENCY.analytics)
  return { range: '30d', points: [/* 30 {day, revenue, visitors} */ [] }
}
```

### 2.2 The instrumented page (the four sections as segments, module 27 rule 4)

The capstone's dashboard structure (from module 25, now with the segments explicit):

```
src/app/(app)/dashboard/
├── layout.tsx            (the session/org reads — the shell's synchronous cost)
├── page.tsx              (renders the four section segments in a grid)
├── revenue/{page? no —}  → the four sections are RENDERED segments:
├── _sections/
│   ├── revenue/{default.tsx (the async section), error.tsx, loading? }
│   ├── orders/{…}
│   ├── activity/{…}
│   └── analytics/{…}
```

(Implementation note: the "sections" are rendered via a shared layout + nested routes (`/dashboard/revenue`, etc.) so each gets its **own `error.tsx`** (module 27 rule 4) *and* its own `Suspense` — the page renders `<RevenueSection />` which is the `revenue` segment's async component. The exact route shape is a module-06 concern (parallel routes, module 09); the lab's point is the *isolation*.)

`FILE: src/app/(app)/dashboard/page.tsx` (lab version — [SERVER])

```tsx
import { headers } from 'next/headers'
import { redirect } from 'next/navigation'
import { Suspense } from 'react'
import { auth } from '@/lib/auth'
import { getActiveOrgId } from '@/services/organizations'
import { RevenueSection } from '@/app/(app)/dashboard/_sections/revenue/default'
import { OrdersSection } from '@/app/(app)/dashboard/_sections/orders/default'
import { ActivitySection } from '@/app/(app)/dashboard/_sections/activity/default'
import { AnalyticsSection } from '@/app/(app)/dashboard/_sections/analytics/default'
import { SectionSkeleton } from '@/components/dashboard-skeleton'

export default async function DashboardPage() {
  const session = await auth.api.getSession({ headers: await headers() })   // shell: ~5ms
  if (!session) redirect('/login')
  const orgId = await getActiveOrgId(session.user.id)                       // shell: ~2ms
  if (!orgId) redirect('/onboarding/select-org')

  return (
    <main className="grid gap-6">
      <Suspense fallback={<SectionSkeleton kind="revenue" />}>
        <RevenueSection orgId={orgId} />
      </Suspense>
      <Suspense fallback={<SectionSkeleton kind="orders" />}>
        <OrdersSection orgId={orgId} />
      </Suspense>
      <div className="grid gap-6 lg:grid-cols-2">
        <Suspense fallback={<SectionSkeleton kind="activity" />}>
          <ActivitySection orgId={orgId} />
        </Suspense>
        <Suspense fallback={<SectionSkeleton kind="analytics" />}>
          <AnalyticsSection orgId={orgId} />
        </Suspense>
      </div>
    </main>
  )
}
```

Each `_sections/*/default.tsx` is the async child (module 27 rule 1): the `await` inside, the boundary outside. The **tracer** (module 19: `src/instrumentation.ts` + the request-scoped log) prints, per request, per service call: `{ t: <ms from request start>, scope: 'revenue', status: 'hit'|'miss', args: {orgId} }` — that log line *is* the lab's measurement.

### 2.3 The measurement protocol (fixed, for every experiment)

1. **Hard load** (a fresh browser tab / `Ctrl+Shift+R` — bypasses the L1 client cache): captures the *server* streaming (flush → fallbacks → resolutions).
2. **Soft navigation** (click from another page — the L1 client cache, module 20): captures the *client* streaming (the RSC payload's holes).
3. For each: record (a) **TTFB** (first byte / shell painted — the dev overlay or the Network panel's first response event), (b) **the fallback flush time** (the skeleton visible), (c) **each section's resolution time** (the tracer's `t` for each scope), (d) **the resolution *order*** (the sequence of scopes in the tracer log).
4. **Warm vs cold**: warm = a second request after a first (the entries are hot in L3); cold = `next build` fresh (or a restart) — *the first* request.

**The lab report template** (`docs/lab-streaming.md` — the deliverable, one table per experiment):

```md
## Experiment N — <name>
Hypothesis: <predicted, from module 27's rules>
Setup: <the variable changed>
Observed:
| event | t (ms) | note |
|---|---|---|
| TTFB (shell painted) | … |
| fallbacks flushed | … |
| activity resolved | … |
| orders resolved | … |
| revenue resolved | … |
| analytics resolved | … |
Resolution order: activity → orders → revenue → analytics
Verdict: hypothesis confirmed / refuted — <the one-line why>
```

## 3. The Experiments

### Experiment 1 — Baseline: the independent sections (the core claim)

**Hypothesis (from the latencies: activity 80 < orders 150 < revenue 300 < analytics 500):** the shell paints first (TTFB ≈ the session/org path, well before 80ms of *section* work); the four skeletons flush in one chunk (they're all "pending" at the first flush); the sections resolve in **completion order — activity, orders, revenue, analytics** — *not* document order (revenue is the first section *visually* but the third to resolve); no section waits for another.

**Run:** the protocol, warm (so the *resolution* latencies are the stub delays, not query costs — the cache hits are ~1ms, the stubs' `delay()` dominates *inside* the entry's execution… wait — a **warm** entry doesn't re-execute (the stub's delay is *inside* the function — the cache skips it). **Correction for the lab:** run Experiment 1 **cold** (a fresh process) — the entries execute the stubs, the delays dominate, the order is *visible*. Then run it **warm** (Experiment 4) — the hits collapse the latencies.

**Expected table (cold):**

| event | expected t |
|---|---|
| TTFB (shell: layout + session 5ms + org 2ms) | ~10–30ms of server work (+ network) |
| fallbacks flushed (all four skeletons — the first `Suspense` hit) | at TTFB (they're part of the first flush) |
| activity resolved | ~80ms + TTFB |
| orders resolved | ~150ms + TTFB |
| revenue resolved | ~300ms + TTFB |
| analytics resolved | ~500ms + TTFB |

Resolution order: **activity → orders → revenue → analytics.** (If your log shows a *different* order, the stubs ran sequentially, not in parallel — find the missing `Suspense` (module 27: a synchronous `await` *between* the sections' renders serializes them — the page's body must not `await` one section before rendering the next boundary; the async children are *independent* precisely because each boundary isolates its await.)

**The check that makes it a lab, not a demo:** look at the **Network panel** — the *one* HTTP response's *content* grows over ~500ms (the streamed chunks: the shell, then the payload + swap scripts). One request, incremental content. (A "soft" implementation — four separate XHRs — would show four requests; this is *one* response, streamed. The module-12 wire format, observed.)

### Experiment 2 — The serializing bug (the anti-pattern, quantified)

**Hypothesis (from module 27's "no boundary around an await" failure):** if the page's body does `const revenue = await getRevenue…` *before* rendering the other sections (a synchronous await in the page — the module-25 anti-pattern "one big boundary" in its purest form: *no* boundaries, a sequential page), the TTFB ≈ **the sum** of the sections (80+150+300+500 ≈ 1030ms) and there are *no* skeletons (nothing to fall back *into* — the page is all-or-nothing).

**Run:** comment out the four `Suspense` wrappers, make the page `await` each section in order (the "naive" version). Hard load. Measure TTFB.

**Expected:** TTFB ≈ 1030ms + network; one paint at the end; the perceived load is *~10×* the streaming version's *last* byte (500ms) — and *20×* its first paint. **Record the ratio** — it's the number for the standup ("streaming made first paint 20× faster, and the last section 2× faster").

**Then** the *middle* anti-pattern (module 27's "one boundary around everything"): keep the awaits sequential but wrap the *whole content* in one `Suspense` — TTFB returns to the shell's cost (the *skeleton* shows early), but the *content* arrives all at once at ~1030ms (one swap, the slowest section's time) — the skeleton is one big block, and the "progressive" perception is *gone* (one pop at the end). **Three configurations, three TTFBs, one last-paint time** — the table of the module's argument.

### Experiment 3 — The nested boundary (module 27's diagram, observed)

**Hypothesis:** with the sparkline (a nested `Suspense` inside the revenue section, stub latency **420ms** — between revenue's 300 and analytics's 500), the **revenue card** (the number) resolves at ~300ms *with its sparkline still a skeleton*; the sparkline resolves at ~420ms *inside the already-visible card*. The card's *content* and its *nested hole* land independently — the nested boundary is a *sub-unit* (module 27's flowchart).

**Run:** add the nested boundary (module 27 §4.1, the `RevenueSparkline`), cold. Measure: the revenue section's resolution (the card) and the sparkline's resolution (the tracer logs the nested scope `revenue.sparkline` as a *separate* line).

**Expected:** `revenue` at ~300ms, `revenue.sparkline` at ~420ms — **the order within a section**: the card before its sparkline (the card's await is *outside* the nested boundary — it's the section's data; the sparkline's is *inside* its own boundary). The **user-visible event**: the revenue *number* is readable at 300ms; the chart draws at 420ms — two events, one section. (The CLS check: the sparkline's skeleton has the chart's *final height* — no jump at 420ms. The module-27 §4.3 rule, verified in the browser: devtools' "layout shift" is 0 for the swap.)

### Experiment 4 — The warm case (caching × streaming, composed)

**Hypothesis (from module 27's "streaming is not caching" + the module-25 contract):** a **warm** request (the entries hot — a second hard load after the cold one) resolves all four sections at **~1ms each** (the L3 hits — the stubs' delays are *skipped*, the cache returns the stored DTOs) — the "last section" lands within **~20ms of the shell** (the 500ms becomes ~10ms). The *streaming* still happens (the boundaries still isolate; the *resolution* is just near-instant) — the warm dashboard is **perceived as "it just appears"** (the shell + the four sections in one or two paints, indistinguishable from a static page — *the point of the whole phase*: the dynamic dashboard, warm, is as fast as a static one, and *stays* dynamic on the cold case).

**Run:** after Experiment 1 (cold), a second hard load (same entries — the same orgId). Measure the four resolutions.

**Expected:** the four `status: 'hit'` lines at ~10–30ms (clustered); the resolution *order* is now **nondeterministic** (all ~1ms — they land in whatever order the reconciler flushes them; the log shows the cluster, not a spread). **The two-waterfall comparison** (the module-27 §8 production exercise's artifact, now with real numbers): cold (spread over 500ms) vs warm (cluster in ~20ms) — side by side, the module-24 contract (warm = instant, cold = graceful), *measured*.

### Experiment 5 — The soft-navigation case (the client-side streaming)

**Hypothesis (from module 27 rule 5 + module 20's L1):** a **soft navigation** *to* the dashboard (the user clicks "Dashboard" from `/orders`) does *not* re-download the page's HTML — it fetches the **RSC payload** for the route; the *shell* of the dashboard renders from the **L1 client cache** (module 20: the client router serves the route's cached shell per its `stale` clock) and the *holes* resolve from the payload (warm: near-instant; the client's stale clock decides whether a *revalidation* check happens — module 20's L1 behavior). The user *sees*: the dashboard's sections appear *instantly* (the L1 shell) — faster than the hard load's TTFB (no server round-trip for the shell at all).

**Run:** from a *different* page in the app (with JS, a soft nav — the link), navigate to `/dashboard`. Network panel: the request is the **RSC payload** (the `RSC: 1` header — the module-12 wire format's client leg), *not* an HTML document. Measure: the dashboard's first paint (it should be *before* the response arrives — the L1 shell — or at the response's first bytes for the holes).

**Expected:** first paint ≈ the click (the L1 shell, the client cache — the *stale* window from module 20: the dashboard's sections were cached in the *client* when the user was last on the dashboard); the *freshness* of the client-cached shell is governed by the `stale` clock (module 22: the `minutes`/`hours` lives' *stale* property — the client serves it without checking for up to that window, then revalidates). **The subtlety to verify:** if the user was last on the dashboard 4 minutes ago (the `minutes` profile's `stale` = 5m → still stale-serving), the soft nav shows the *client-cached* shell (4 minutes old) and *then* the server's fresh payload updates the changed sections *in place* (the swap — the user sees a section's numbers *tick* to fresh — the "it updated" flash, a *feature*: the data is visibly fresh, not silently stale). If >5m, the client *revalidates first* (a check request) — the soft nav is slightly slower (the revalidation round-trip) but the shell is fresh. **Record both** (fresh-shell case and revalidating case) — the module-20 L1 row, *observed* on the dashboard.

### Experiment 6 — The per-section error isolation (the reliability half)

**Hypothesis (from module 27 rule 4):** if the **analytics** service throws (a 500 — the stub made to reject), the **analytics section** shows its `error.tsx` (the "couldn't load" + Try again, module 27 §4.2) *while the other three sections are fully rendered and interactive* (they streamed first — they're *already painted*; the error is *isolated* to the segment's boundary). The *page* does not crash (no full-page error, no `global-error`); the *section* is the unit of failure.

**Run:** the analytics stub rejects (`throw new Error('analytics db down')`) — or better, the *real* failure shape: the service throws an `AppError` (module 17) that the *section* doesn't catch → it propagates to the segment's `error.tsx`. Hard load. Observe the three good sections (interactive — the orders' cancel button works), the analytics error card, and the **digest** (the `error.digest` in the telemetry POST — module 09-04's incident ID — module 27 §4.2's `useEffect`).

**Expected:** three sections rendered + one error card; the `Try again` calls `reset()` (the boundary re-renders the segment — the analytics stub, now succeeding (toggle the reject off), resolves on retry); the *other* sections' state is *untouched* (the reset re-runs only the *errored segment's* render — the module-09-04 error-boundary scope, verified: the reset does *not* re-run the revenue/orders/activity entries (the tracer shows *no* re-execution for them — they're *not* part of the errored segment). **The reliability claim, evidenced:** one section's 500 is a *section* incident, not a *page* incident — the user loses one card, not the dashboard.

### Experiment 7 — The CLS audit (the streaming tax, measured)

**Hypothesis (from module 27 §7):** every section's **swap** (skeleton → content) produces **zero layout shift** (the skeleton's geometry = the content's geometry — module 27 §4.3). Devtools' **Layout Shift** total for the *cold* load ≈ **0** (the only shifts: none — the skeletons are the *placeholders* at final size).

**Run:** cold load, the Performance panel's "Layout Shift" track (or the Lighthouse run's CLS). Compare: with the skeletons (expect 0) vs a **control** (the skeletons replaced by `<div>Loading…</div>` — *wrong-geometry* fallbacks — expect a *visible* CLS: each section's swap *pushes* the content below it down; the total is the sum of the four sections' height differences).

**Expected:** skeletons → CLS ≈ 0 (the dashboard is *stable*); "Loading…" divs → CLS ≈ the four sections' combined height delta (the "page jumps" bug, *quantified* — the number the module-27 tax line was about). **The design-system rule, proven:** the skeleton *is* the CLS control (module 13's registered skeletons, with real geometry, are *load-bearing*, not decorative).

## 4. The Lab Report (the deliverable — `docs/lab-streaming.md`)

The seven experiments' tables (the §2.3 template), **plus the synthesis section** (the phase-gate claim, now evidenced):

```md
## Synthesis — the dashboard's streaming contract, measured
1. TTFB (hard, cold): <measured> — the shell's cost (session/org), NOT the sections' sum.
   (Claim: < 100ms server work — verified/refuted.)
2. Independent sections: resolution order = completion order (Exp 1) — <order observed>.
   The serializing bug's cost: <Exp 2's ratio>× first-paint slower.
3. Nested boundaries: <Exp 3's card/sparkline split> — sub-units land independently.
4. Warm vs cold (the two waterfalls): cold = <spread, ms>; warm = <cluster, ms>.
   The module-24 contract (warm = instant, cold = graceful): <evidenced>.
5. Soft nav (L1): shell from the client cache (<the stale-window behavior observed>);
   the freshness update is a visible in-place swap (the "tick" feature).
6. Error isolation: <Exp 6 — one section's 500, the other three untouched, the digest logged>.
7. CLS: skeletons → <measured>; control → <measured> — the skeleton is the CLS control.
Verdict: the phase's claim stands — <the one paragraph: what the numbers show,
what surprised you (every lab has one), what the next phase changes>.
```

**The "what surprised you" is required** (the lab's honesty rule): if a hypothesis was *refuted* (a section landed in the wrong order — find the missing boundary; the warm cluster was wider than 20ms — find the entry that didn't hit), the report *documents the refutation and the cause* — a lab that confirms everything is a demo, not a lab.

## 5. Production Code — The Real Services (the lab's second run)

**The lab's final act:** swap the stubs for the **real services** (the module-25 document's functions: `getRevenueSummaryCached`, `getRecentOrdersCached`, `getRecentActivityCached`, `getAnalyticsCached` — the `'use cache'` + profiles + tags, the real Drizzle queries) and **re-run Experiments 1, 4, and 6** (the three that change with real data):

- **Experiment 1 (real):** the cold-case latencies are now the *real* query times (the DB round-trips — typically 5–50ms per section on the capstone's data, *faster than the stubs* — the stubs' 80–500ms were *pessimistic*; the real cold case is *also* graceful (skeletons still earn their keep at 5–50ms? — the lab's honesty: *sometimes the skeleton is a flash* (a 10ms resolution beats the skeleton's first paint — the "pop" is *perceptually free* at <100ms; the skeleton's job is the *cold-deploy* and *high-latency* cases, not the 10ms warm-local case). **Record the real numbers** — the stub lab *predicted the shape*; the real run *sets the actual values*.
- **Experiment 4 (real):** the warm cluster is *even tighter* (the real hits are ~1ms — the DTOs stored in L3) — the "instant" dashboard, *for real*.
- **Experiment 6 (real):** the real failure (a query that 500s — e.g., a *missing* analytics table, or an org with no analytics data that the service mis-handles → the `AppError`'s 500 path) — the same isolation, the real error path (the module-17 `AppError`'s status → the `error.tsx`'s message mapping, module 09-04's `error.digest`).

**The two runs' comparison is the lab's final table:** stub (controlled, predicted) vs real (measured, actual) — the *shape* matches (the order, the isolation, the warm/cold contract); the *values* differ (the real queries are faster; the real cold-deploy is the one that *needs* the skeletons — a fresh `next build` with cold L3 on a *real* Postgres is the production cold case, and its latencies are the stubs' job: the stubs *approximated the production cold case*, which the local dev's fast DB doesn't have).

## 6. Common Mistakes (the lab's failure modes — what a *bad* lab looks like)

| Bad lab | The tell | The fix |
|---|---|---|
| Measuring only the warm case | The "instant dashboard" claim with no cold numbers — the *cold-deploy* incident (module 24's "cold holes are your real latency") goes unmeasured | Both cases, every time (the protocol's §2.3, the warm/cold split) — the cold case is the *honest* one (production's first request post-deploy is cold) |
| Soft-nav measurements with the L1 cache *disabled* (incognito + hard reload every time) | The "soft nav is instant" claim untested — the L1 behavior (module 20's client cache) is the *point* of the soft case | The protocol's two load types, *both* — a hard load (server streaming) and a soft nav (the L1 shell + the RSC payload) are *different measurements of different things*; mixing them conflates them |
| The tracer not logging *per-entry status* (hit/miss) | The warm/cold distinction is *invisible* in the log (you can't tell a hit from a miss) — the module-19 tracer's `status` field is the lab's *whole* warm/cold evidence | The tracer logs `status: 'hit' \| 'miss'` per entry (the module-19 instrumentation, verified before the lab — a lab on an uninstrumented app measures *nothing*) |
| Comparing the stub run's *values* to the real run's *values* as if they should match | "The real dashboard is 30× faster than the lab — the lab is wrong" — no: the stubs *modeled the production cold case* (high latency), the local dev's DB is *not* production (the module-18-01 "measure where it runs" line) | The stub run's *shape* (the order, the isolation, the contract) is the prediction; the real run's *values* are the local-truth; the *production* values come from the module-18-01 RUM/trace (the lab's numbers are *dev-environment* numbers — label them as such in the report) |
| Skipping Experiment 2 (the anti-pattern) | The lab shows the streaming works, but not what it's *worth* (the ratio is the *business case* — the module's whole argument, quantified) | The anti-pattern run is the *control* — a lab without its control can't attribute the win |
| The CLS audit with the Performance panel *closed* | "CLS is fine" by eye — the *jump* is visible but unquantified (the design-system rule, unaudited) | The Performance panel's Layout Shift track (or Lighthouse) — the number, not the impression |

## 7. Security & Performance Notes (the lab's findings, labeled)

- **The streamed HTML is public to the *viewer of that request*** (the module-27 §6 line, lab-verified): the dashboard's streamed content is *session-scoped* (the session cookie gated the *request* — the module-25's shell) — a *different* user's browser never receives this user's streamed sections (the gate is the *request*, not the *render*). **Verify it (the lab's security check):** two browser contexts, two sessions, two dashboards — the streamed *content* differs per session (the tracer's `args: {orgId}` line, per context — the orgId is *different*; the streamed HTML carries the *right* org's data, per request). This is the *multi-tenant* streaming check (module 11-02's cross-tenant test, applied to the *stream*, not just the data).
- **The RSC payload (soft nav) is the module-12 wire format's second part** (lab-verified in Experiment 5's Network panel): the payload is *not* JSON (it's the RSC flight format — the module-12's two-part format, the client leg) — and it's **not inspectable as data** (it's a *render* format, the module-14's serialization rules apply — a hole's payload carries the *DTOs*, module 14; a non-serializable value in a *hole* fails at the *swap* (the module-27 §6 late-stage failure) — the lab's Experiment 4 (warm) is where such a bug *shows up* (the cold run renders the fallback, the warm run *swaps* — the swap is where the serialization is *exercised*).
- **Performance, the lab's standing numbers** (the report's synthesis, the module-18-01's *first* data points — the performance phase's baseline is *this lab*): the TTFB, the resolution spread (cold/warm), the soft-nav shell time, the CLS — **the performance phase (18) measures these in *production-like* conditions; the lab measured them in dev.** The handoff: the lab's *instruments* (the tracer, the protocol) are the module-18-01's, and the lab's *numbers* are its *dev* baselines.

## 8. Exercise (the phase-gate, the lab itself)

**The gate is the lab** (modules 27's rules + 28's evidence, applied to the capstone dashboard):

1. **Build the lab bench** (§2): the stubs, the instrumented page (the four sections as segments with their own `error.tsx` + `Suspense`), the tracer logging per-entry `status` (hit/miss) + `t` + scope.
2. **Run all seven experiments** (the protocol, §2.3), cold *and* warm where the experiment says so, hard *and* soft where it says so.
3. **Write the lab report** (`docs/lab-streaming.md`, §4's template): the seven tables + the synthesis (the phase-gate claim, evidenced) + the required "what surprised you."
4. **The real-services run** (§5): Experiments 1, 4, 6 with the real services (the module-25 document's functions) — the two-runs comparison table.
5. **The security check** (§7): the two-context multi-tenant stream verification (the per-session content, the tracer's per-orgId lines).

**The exit criteria (the phase gate, the roadmap's "Dashboard streams all sections independently"):**
- [ ] The shell's TTFB is measured *and* is the session/org cost (not the sections' sum) — Experiment 1 + 2's ratio, recorded
- [ ] The four sections resolve in **completion order** (independent) — Experiment 1's order, recorded
- [ ] The nested boundary (the sparkline) lands *inside* the already-visible card — Experiment 3
- [ ] The **warm/cold two-waterfalls** exist, side by side, with the contract (warm = instant, cold = graceful) *stated against the numbers* — Experiment 4 + 1
- [ ] The soft-nav L1 behavior is observed *and* described (the client shell + the in-place freshness swap) — Experiment 5
- [ ] One section's 500 is a *section* incident (the other three untouched, the digest logged) — Experiment 6
- [ ] CLS ≈ 0 with the skeletons (the control, without them, is measured) — Experiment 7
- [ ] The real-services run's numbers are recorded *and* labeled (dev-environment values, the production baselines are module 18's)
- [ ] The report's "what surprised you" is written (at least one finding — if it's empty, the lab wasn't honest)
- [ ] **Capstone stage after Phase 6 (the roadmap):** "Dashboard streams all sections independently" — *true, measured, in `docs/lab-streaming.md`*

## 9. Architecture Challenge

**Prompt:** The dashboard's **Analytics** section is *slow in production* (a 2s query — the 30d rollup over a large org) but *fast locally* (the 50ms stub / the small local DB). The lab (dev) says "graceful, 2s is fine (the skeleton)" — but the *production* RUM (module 18's future measurement) will show the analytics section's 2s as the *worst* section (the user stares at its skeleton for 2s on every cold load).

Without *pre-computing* the rollup (a nightly job — the module-25 document's "the nightly job is the primary refresher" — is *forbidden* here: assume it doesn't exist yet), design **three** fixes, ranked, each: the mechanism (what changes — the query? the cache's life? the boundary's placement? a *progressive* decomposition of the analytics section itself?), the cold-case latency it produces, the warm-case cost (the regeneration it adds), and the *perceived* effect (what the user sees). Then: which one do you ship *this week* (the smallest change, the biggest perceived win), and what's the *architecture* answer (the one that's "right" for the scale, even if it's a bigger build)?

<details>
<summary>Model answer</summary>
The constraint (no pre-compute) means the 2s is *in the query* (the 30d rollup over a large org) — the fixes attack: (A) the *query's* cost, (B) the *section's* decomposition, (C) the *cache's* shape.
**Fix 1 (ship this week — the progressive decomposition, the smallest change, the biggest perceived win): split the analytics section into *two nested boundaries* (module 27's nested-boundary rule, applied to *decomposition*):** the **headline** (the 30d *totals* — the two numbers: revenue, conversion — the *cheap* aggregation, ~200ms) in the *outer* part of the section; the **charts** (the 30d *series* — the *expensive* part, the 2s) in a *nested* `Suspense` (the §4.1 sparkline pattern, at the section level). **Cold case:** the headline lands at ~200ms (the user sees *real numbers* in 200ms — the "it's loading" is *gone* for the *decision* data); the charts land at ~2s (the skeleton *within* the already-answered section — the perceived cost of 2s is *halved*: it's "the chart is rendering" not "the whole section is empty"). **Warm case:** both are cached entries (the headline: `hours` · the charts: `hours` — the same data, two views of it — the module-25's view-args-join-the-key rule: the headline and the charts are *two entries* (two functions) sharing the *same* tag (`'analytics:{orgId}'` — the data's tag, module 23's rule: views multiply entries, tags invalidate truth) — the warm case is *unchanged* (both hits, ~1ms). **Perceived:** the section *answers* in 200ms; the *richness* (the charts) arrives when it's ready — the module-27's nested boundary, doing its job (the sub-unit lands into the already-visible content). **This is the ship-now answer** (a boundary + a split of the service into two functions (the headline query is *cheaper* — it's the *totals*, not the *series* — the query cost is split, not just the *perception*): total cost is roughly the same (the series still runs, but *in parallel with the user reading the headline* — the *perceived* wait is 200ms, not 2s), warm cost is +1 entry (negligible), and the perceived win is *the whole 2s* (the user is *looking at the numbers* while the chart renders).
**Fix 2 (the architecture answer, the scale one — the materialized rollup, *not* a nightly job but a *write-time* aggregate):** the query is slow because it *aggregates per read* (the 30d rollup over raw orders/traffic rows) — the architectural fix is a **materialized aggregate table** (`analytics_daily(orgId, day, revenue, orders, visitors)` — the *rollup*, maintained *at write time* (the order-paid event appends/updates *one row* per (org, day) — O(1) per mutation, module 23's invalidation touches it via the *existing* `'analytics:{orgId}'` tag)): the read is now a *range scan over 30 rows* (~10ms, *not* a 2s rollup) — **cold case: 10ms** (the skeleton is *gone*; the section is as fast as the others); **warm case: unchanged** (the entry still caches the DTO — now *faster* to fill); **perceived: the 2s disappears** (the decomposition (Fix 1) is *unnecessary* — the section is fast end-to-end). **The cost:** the write path (the order-paid event now also updates the daily row — a *transactional* write (module 09-04) — the module-23's "the webhook is a mutation" rule, extended: the analytics aggregate is *written* by the mutations, not *read* from them); the backfill (the historical 30d is computed *once* when the table is introduced — a one-off job, *not* the forbidden nightly re-rollup — the *incremental* maintenance is write-time, O(1)). **This is the "right" answer at scale** (the module-25 document's "nightly job" is the *cheap* version of this — but the *write-time* aggregate is the *correct* one: the read is O(rows-in-range), not O(rows-in-range × the rollup cost) — at 30d × a large org, the write-time aggregate is the *architecture*, and the nightly job is the *compromise* the document named for when the write-path cost was too high).
**Fix 3 (the middle — the *cache's* shape, if the query stays):** the analytics entry's *life* is `hours` (the module-25 document) — the *cold* case (post-deploy) is the 2s; the *warm* case is ~1ms. The fix: **make the cold case rare** — the *deploy* is what colds it (the module-24's "build ID in the key = global invalidation") — so the answer is: **a `remote` cache (the module-21's `'use cache: remote'` / a Redis tier) for the analytics entry** (the entry *survives* the deploy — the build-ID key doesn't *fully* cold a *remote* entry (the module-21's remote tier's durability — the deploy's new buildId *misses* the in-memory tier but *hits* the remote (the key's buildId… — the nuance: the module-21's key contract *includes* the buildId (the deploy *does* cold the remote too) — so the honest Fix 3 is: the remote tier doesn't *survive* the deploy (the buildId), it *survives the *instance* (the serverless multi-instance cold, the module-25's "per-instance cold hits" note) — for a *single-instance Docker* deployment (the capstone's baseline, module 22-01), the in-memory tier *already* survives (the long-lived process) — the deploy's restart is the *only* cold; the *remote* tier's value is the *serverless* scale case (many instances, the module-21's remote rule). **So Fix 3 is scale-conditional** (the remote tier is the *serverless* answer, not the *Docker* answer) — and it doesn't fix the *first-ever* read for an *org* (the new org's first dashboard is cold regardless — the 2s, once, per org, per deploy).
**The ranking (perceived win / effort / correctness):** Fix 1 (the decomposition) *now* (a boundary + a service split — *this week*, the perceived 2s → 200ms, the warm cost is +1 entry); Fix 2 (the write-time aggregate) *this quarter* (the *architecture* answer — the 2s → 10ms *end-to-end*, the skeleton *retired* for the section, the write-path cost is the O(1) per-mutation update (the module-23's invalidation path, extended)); Fix 3 (the remote tier) *when* the deployment is serverless (the multi-instance cold, the module-21's rule — not this week, the Docker baseline doesn't need it). **The generalization (the lab's standing lesson):** *a slow section is attacked in order of *perceived cost*: (1) *decompose* it (the cheap data first, the expensive data in a nested boundary — the user *sees* the answer early, the richness arrives late — module 27's nested boundary, applied to *decomposition*); (2) *make the query cheap* (the aggregate, the write-time rollup — the *architecture*, the data's shape is the *real* fix — the query's cost is a *data-model* problem, not a *rendering* one); (3) *make the cold case rare* (the cache's tier — the *deployment-conditional* fix, the module-21's remote rule). **Decompose first (perception), aggregate second (reality), tier third (scale)** — the order is the *perceived cost* order, and it's the order the user *feels* first.**
</details>

## 10. Official Documentation

- Streaming UI (React, the underlying model): https://react.dev/reference/react-dom/server/streamableHTML
- Suspense (Next): https://nextjs.org/docs/app/api-reference/suspense
- Fetching Data (async/await, streaming): https://nextjs.org/docs/app/getting-started/fetching-data
- Loading & Skeletons (React 19): https://react.dev/reference/react-dom/server/streamableHTML
- Handling Errors (the error files, the reliability half): https://nextjs.org/docs/app/guides/handling-errors
- Caching (the holes' warm case): https://nextjs.org/docs/app/getting-started/caching

## 11. What You Should Know Before Continuing (Phase 6 complete — the gate)

- [ ] I can state the streaming model *precisely* (TTFB = the shell; holes land by completion; streaming ≠ caching — they compose)
- [ ] I know the five placement rules and can *see* their violations in a tracer log (the serialization, the missing boundary, the over-boundary pop)
- [ ] I've **run the seven experiments** and the lab report (`docs/lab-streaming.md`) exists with the two-waterfalls, the ratio, the CLS, and the "what surprised you"
- [ ] The warm/cold contract is *measured* (not just stated) — the phase's standing claim, evidenced
- [ ] The per-section error isolation is verified (one 500, three untouched, the digest logged)
- [ ] The multi-tenant stream check is done (the per-session content, the per-orgId tracer lines)
- [ ] **The phase gate (the roadmap):** "Dashboard streams all sections independently" — *true, measured, in the report*
- [ ] I know the lab's numbers are *dev* baselines (the production measurements are module 18's) — the handoff, labeled

**Next phase:** Phase 7 — **Server Functions / Actions (mutations)**: the `'use server'` API in depth (the module-23 tails get their head): the Server Functions vs the (legacy) "Server Actions" terminology, the mutation path end-to-end (the POST-only dispatch, the sequential semantics), forms + progressive enhancement, the Zod validation round-trip, the optimistic UI + pending states, and the Action vs Route Handler vs external API decision matrix. (The capstone's mutation inventory — the module-23 invalidation specs' *heads* — is built here.)
