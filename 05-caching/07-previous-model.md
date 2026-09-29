# Module 26 — The Previous Model (and Migrating From It)

**Phase 5: Caching · Module 26 of 101 · the final module of Phase 5**

> **Where does this run?** The "Previous Model" (pre-15.2) `[SERVER]` data path: `fetch` with `next: { revalidate, tags }`, the request memoization, `force-static`/`force-dynamic`, the fetch cache keyed by URL. You'll meet it in legacy codebases, team members' muscle memory, and your own migration. This module is the **translation table** — and the migration playbook.

---

## 1. Concept — How the old model worked (accurately)

The Pre-15.2 data cache ("the fetch cache") had one rule: **`fetch` calls in the data path were cached, keyed by URL + options**, with behavior governed by:

| Mechanism | What it did | The limitation that motivated its replacement |
|---|---|---|
| `fetch(url, { next: { revalidate: 3600 } })` | Time-based: cache the *response* for an hour (ISR-like, per URL) | The life was on the **fetch call** — invisible from the component that *uses* the data; two components fetching the same URL with different options = different entries (or a dedup that surprised you) |
| `fetch(url, { next: { tags: ['products'] } })` + `revalidateTag('products')` | Tag-based invalidation (the part that *survived* — `revalidateTag`/`revalidatePath` still exist) | Tags were on fetch calls, not on the *logic*; invalidating a tag re-fetched the URLs, it didn't re-run any *computation* you'd written |
| `revalidatePath('/products')` | Re-prerender the route | The route is the unit — over-broad, always |
| Request memoization | Dedup identical fetches *within one request* | Fine — this still exists (module 20's L5) |
| `cache = 'force-cache'` (undici) | The only way to opt a fetch into the cache (the default `no-store` gotcha) | The cache was *opt-in per fetch* — the mental model was "remember to cache the fetch," which teams did inconsistently |
| `cookies()`/`headers()` in the tree | Made the *whole route* dynamic | The per-route binary (module 24): one `cookies()` read for a "Hello, X" killed the page's static rendering |
| `export const dynamic = 'force-static'` / `force-dynamic` | Per-route overrides | The per-route unit, again |
| `unstable_cache(fn, key, opts)` | Manual function caching (unstable for years) | The escape hatch that teams built on before it stabilized — the direct ancestor of `'use cache'` |

**The old model's mental image:** a *response cache* around `fetch` (a poor man's CDN-in-process) + per-route static/dynamic fate + tag invalidation that only reached *fetches*. The data **logic** (the query orchestration, the DTO mapping, the service) was *never* a caching unit — only the raw fetches were.

**Why it was replaced** (the honest engineering reasons, so you can argue them):
1. **The unit was the wrong one.** Caching a *fetch response* doesn't cache your *service logic* (the aggregation, the joins, the per-tenant scoping) — a service doing 3 fetches + a DB call was 90% uncacheable no matter the fetch options. The correct unit is the *function* (`'use cache'`).
2. **The life was invisible.** `revalidate: 3600` buried in a fetch call is a secret; a profile on the scope is a *document* (module 22's thesis: the lifetime is a contract, and contracts are visible).
3. **The route was the wrong granularity** for dynamic parts (`cookies()` kills the page) and the wrong unit for *freshness* (path revalidation over-invalidates).
4. **Opt-in caching is inconsistent** (`force-cache` everywhere, forgotten in the new code). Opt-out (caching is the default inside `use cache`) is a *policy*.

**What survives (the continuity table — what does NOT need re-learning):**

| Survives into the new model | Notes |
|---|---|
| `revalidateTag(tag)` / `revalidatePath(path)` / `revalidate`/`tags` *semantics* | Tag invalidation is the same idea, now applied to *entries* (the framework's API kept its name; `revalidateTag` gained the `profile` argument — module 23) |
| Request memoization (dedup within a request) | L5 in module 20's stack — unchanged |
| `revalidatePath` as the escape hatch | Still there, still broad, still use tags when you can |
| The *idea* of ISR (stale-while-revalidate) | Now a per-entry property (`revalidate`) instead of a route's fetch-option |
| **Tags as a design discipline** | The typed-tags module (module 23) works identically in both models |
| `next/fetch`'s `force-store`/`force-no-store`? | Gone in the new model — the directive is `'use cache'` on the *scope*, not a cache-mode on the fetch. (If you see them in old code, that's the fetch-cache era.) |

## 2. Mental Model — the translation table (the module's core artifact)

**Old → New, line by line:**

| Old-model code | New-model equivalent | The change in meaning |
|---|---|---|
| ```ts
const res = await fetch(url, { next: { revalidate: 3600 } })
const data = await res.json()
// …transform inline in the component
``` | ```ts
export async function getProductCached(slug: string) {
  'use cache'
  cacheLife('hours')
  cacheTag(TAGS.product(slug))
  return transform(await (await fetch(url)).json())   // the transform is INSIDE the entry
}
``` | The **transform joined the cache**: the entry is the *DTO*, not the raw response. This is the biggest conceptual jump — caching your *logic's output*, not a HTTP response. |
| ```ts
const res = await fetch(url, { next: { tags: ['products'] } })
``` | `cacheTag(TAGS.products)` inside the scope | Tags on the scope, not the fetch — and they mark the *entry* (the function's output), so invalidating them re-runs the *whole service* |
| `revalidateTag('products')` | `revalidateTag(TAGS.products)` (+ `updateTag` for actors — the *new* primitive) | Same primitive, same meaning; the new model adds the actor/fleet split (module 23) |
| `export const dynamic = 'force-dynamic'` (route) | No route-level switch: the route's *reads* decide — uncached/request-scoped reads make the *sections* holes (module 24) | The per-route binary dissolves into per-read decisions; "force dynamic a page" is now "make its reads uncached" — a *weaker, more surgical* operation |
| `cookies()` read in a component (page went dynamic) | Read the session **outside** the cached scopes (request-scoped), pass values in as args (module 21's runtime-API rule) | Same *pattern* (runtime data outside the key), now *enforced* (a runtime-API read inside a `use cache` scope is a **build error**, not a silent route-dynamization) |
| `unstable_cache(fn, [args], { revalidate })` | `'use cache'` + `cacheLife` on the function | The unstable escape hatch became the stable default — the key is now `buildId + functionId + args` (you can't build a *custom* key anymore; the args ARE the key — the module-21 contract) |
| The fetch cache (response-level, keyed by URL) | **Gone** (in `cacheComponents` apps): caching is at the *function* level (L3) + the request memoization (L5). A raw `fetch` inside a `use cache` scope is cached *as part of the entry* (running once per entry); outside, it runs per call | The response cache as a *general* layer was removed — the layers it covered are now explicit (function entries + memoization). A plain `fetch` outside a scope is *uncached* (like `no-store`) — the default is no longer "surprised by caching" |

**The one-sentence translation for a team:** *"Stop caching fetches; start caching functions. Everything that was a fetch option is now a scope attribute; everything that was a route fate is now a per-read decision."*

## 3. Architecture — the migration playbook (a real legacy app, step by step)

**The legacy codebase (a typical 2024 SaaS dashboard on Next 14):**

```
app/dashboard/page.tsx
  - 'use server'? no — pages read fetches inline
  - cookies() at the top → whole page dynamic
  - 4 fetches with { next: { revalidate: 60 } } + inline .json() + inline mapping
  - a <ClientRefresh> island polling the dashboard's API route every 30s (the old "live" trick)
  - API route /api/dashboard (a BFF the client polls) — the "old model's" data path
app/api/dashboard/route.ts
  - the same 4 fetches, re-copied, re-mapped (the service logic, duplicated for the client)
```

**Step 0 — Decide the target** (the module's gate): migrate to the Cache Components model *if* the app's pain is (a) per-request rendering of mostly-stable data, (b) duplicated service logic (server + BFF), (c) inconsistent caching. The capstone is built the new way from day one; a legacy app migrates. (Both are in scope for this course — you're being hired for one or the other.)

**Step 1 — Extract the services (the highest-value, lowest-risk step).**
Move the 4 fetches + mappings out of the page *and* the API route into `src/services/dashboard.ts` — one function per section (module 17's rules: orgId-first, DTO-out, `AppError`). **Both** the page and the API route now call the *same* services (the self-API anti-pattern, module 16, dies here). This step changes *no* caching behavior — it's a pure refactor. Ship it alone.

**Step 2 — Add the directive, read the key, write the profile.**
`'use cache'` on each service function + `cacheLife` (the profile from the module-22 procedure) + `cacheTag` (the typed tags). **Before flipping `cacheComponents: true`**, the app is still on the previous model: the directive is *ignored* (it's a no-op until the flag) — so this step is *documentation-as-code* (the lives and tags exist in the code, auditable) with zero behavior change. (On 15.x, `cacheComponents` was the `experimental.cacheComponents` flag; on 16 it's stable — module 02's setup.)

**Step 3 — Fix the key violations (the build is your auditor).**
Flip `cacheComponents: true`. The build now *enforces*: runtime-API reads inside scopes (the page's `cookies()` at the top, if it leaked into a cached path) are errors; unserializable outputs (a `Date` object in a DTO — module 14) are errors. Fix each: session reads move to the layout/page (outside), DTOs become plain objects. **This is the cheapest possible moment to do the module-14/21 discipline** — the build tells you exactly where, instead of a production incident.

**Step 4 — Kill the polling island (the UX upgrade).**
The `<ClientRefresh>` 30s poll is deleted: the sections are now cached entries with `minutes` lives (the module-25 document) — the *next soft navigation* to the dashboard shows fresh data (the client router re-validates per the stale clock, module 20's L1), and a mutation's `updateTag` re-renders the page *immediately* (module 23). If a true "live" need exists (the orders page's "just changed" feel), it's an SSE upgrade (module 18's challenge) — *not* a resurrected poller.

**Step 5 — Delete the API route (or repurpose it).**
The BFF's reason for existing (duplicating service logic for the client) is gone — the client gets data via the RSC payload (module 12's wire format). The `/api/dashboard` route: deleted, unless a *third party* (a mobile app, a partner) genuinely needs the HTTP shape — in which case it's a *real* API (module 08-04) calling the *same* services (the shared seam, module 17). The deletion checklist (module 16) applies, line by line.

**Step 6 — Audit the invalidation (the module-23 audit).**
For every mutation: the tags it invalidates, the actor/fleet split, the write→invalidate order. The typed-tags module + the inventory (module 20) + this audit = the caching *system* is now documented and owned — the migration's final deliverable.

**The migration's risk profile, honestly:** Steps 1–2 are safe (no behavior change). Step 3 is where things break (the key violations) — and it's where the build *helps* (errors, not mysteries). Steps 4–5 change *user-visible* behavior (the polling stops; data freshness is now governed by the document) — **staging + the module-19 request-log diff** (same request, old vs new: what renders, what's cached, what's streamed) is the verification. Step 6 is continuous (every new mutation writes its spec).

## 4. Production Code — the before/after, one function

**BEFORE** (legacy, inside the page — the 2024 pattern, exactly as it was written):

```tsx
// app/dashboard/page.tsx (legacy Next 14 — [SERVER] component, whole page dynamic)
import { cookies } from 'next/headers'

export default async function DashboardPage() {
  const session = await getServerSession()          // → cookies() → page is DYNAMIC (old model)
  const { orgId } = session
  const [revenueRes, ordersRes, activityRes, analyticsRes] = await Promise.all([
    fetch(`${API}/revenue?orgId=${orgId}`, { next: { revalidate: 3600, tags: ['revenue'] } }),
    fetch(`${API}/orders?orgId=${orgId}&limit=10`, { next: { revalidate: 60, tags: ['orders'] } }),
    fetch(`${API}/activity?orgId=${orgId}`, { next: { revalidate: 60, tags: ['activity'] } }),
    fetch(`${API}/analytics?orgId=${orgId}&range=30d`, { next: { revalidate: 3600, tags: ['analytics'] } }),
  ])
  const revenue = (await revenueRes.json()).data    // mapping INLINE, per request, per component
  const orders = (await ordersRes.json()).items
  // …4 sections, each with its own .json() + its own mapping + its own error handling
  return (/* … */)
}
// And /api/dashboard/route.ts does the SAME 4 fetches + mappings for the polling island.
// The 3600 on revenue is a SECRET: no one can see it from the component. The page is
// dynamic because of the session. The polling island re-fetches all 4 every 30s.
```

**AFTER** (new model — the module-25 page, the same dashboard):

```tsx
// src/app/(app)/dashboard/page.tsx ([SERVER] — session-scoped, sections are cached entries)
const session = await auth.api.getSession({ headers: await headers() })   // request-scoped (outside)
if (!session) redirect('/login')
const orgId = await getActiveOrgId(session.user.id)                       // request-scoped

<Suspense fallback={…}><RevenueSummary orgId={orgId} /></Suspense>        // ← cached entry
// where getRevenueSummaryCached = 'use cache' + cacheLife('hours') + cacheTag(TAGS.revenue(orgId))
// — the life is VISIBLE, the key is (orgId), the invalidation is TAGGED, the mapping is INSIDE the entry,
//   the poller is GONE, and the BFF route that duplicated it is DELETED.
```

**The diff is the whole migration**, function by function: *response cache → function cache, secret life → visible profile, inline mapping → DTO at the seam, polling → tags, BFF duplication → one service.*

## 5. Common Mistakes (the migration's specific failures)

| Mistake | Symptom | Fix |
|---|---|---|
| "Just flip the flag and fix what breaks" | Step 3's violations surface as *production* failures (silent staleness, not errors, on the paths the build can't see) | The build *is* the auditor — but only for what it can analyze: the *dynamic* paths (request-time code the build can't trace) need the module-19 request-log diff in staging |
| Keeping the polling island "until the migration is proven" | Two freshness stories run in parallel; the island's 30s poll *masks* the cache's behavior (you can't see whether the cache works) | Delete the poller in the same PR as the flag flip — or you've measured nothing |
| Migrating one function, leaving the rest on fetch-options | The two models *coexist* in one app: the entries (new) and the response cache (old) both run — double caching, double invalidation surfaces, double the mystery | The flag is per-APP (there's no per-function toggle): either the app is on the new model or it isn't. Migrate all the fetch-cache calls in one release (steps 1–3), not a trickle |
| `revalidateTag` from the action, but the entry's tag was on the *old* fetch and you forgot to add `cacheTag` in the new function | The invalidation is a no-op (the entry has no tag) — the classic "I invalidated but it's stale" | The module-23 audit (tags → readers) catches the orphan invalidation; the typed-tags module makes it grep-able |
| Porting `force-dynamic` routes "as-is" (making everything uncached in the new model) | The migration *lost* the caching it was supposed to gain (the route was dynamic because of ONE `cookies()` read; now the whole route's data is uncached) | The module-24 decision table, per read: the session stays request-scoped, the *data* gets profiles. The old route's fate was one read's fault — don't inherit it |
| Assuming `revalidate: 3600` on the old fetch "means the same thing" as `cacheLife('hours')` | It doesn't: the old revalidate was *on the response* (the JSON, before your mapping); the new life is *on the entry* (your DTO, after the mapping) — and the old model re-fetched the URL (cheap, cached elsewhere); the new one re-runs the *service* (the full query) | Budget the regeneration cost (module 22's storm note): entries re-run logic, responses didn't. Stagger lives; the cost is real but usually small (one indexed query) |

## 6. Security Notes

- **The migration is a security review** (the old model's secrets become the new model's build errors): `cookies()` scattered through components was a *latent* tenancy risk (a `cookies()` read in a component that later got `'use cache'` = a public entry of user data). Step 3's build errors surface every one. **Treat the Step-3 error list as a security triage list**, not an annoyance list — each fixed violation is a potential cross-user leak closed.
- The old model's *response cache* was keyed by URL *only* — `fetch(`${API}/revenue?orgId=${orgId}`)` cached the response per URL, which *included* the orgId (the query param was in the key — accidentally safe). The new model's key is the *args* (also includes orgId — deliberately safe). But the *mapping moved inside the entry*: a legacy bug where the mapping *assumed* the caller's org (read a global instead of using the orgId param) was invisible in the old model (the response was per-URL; the bug was downstream) and becomes *entry-level* in the new model (the poisoned DTO is cached and served to the key's owner — the bug is still a bug, but now it's *cache-poisoning*: the wrong data is stored, not just computed). The service contract (orgId as the *first* param, module 17's rule 1) is the control — and the module-11-02 cross-tenant tests are the migration's *final* gate.
- Deleted BFF routes: confirm no **third party** (mobile, partners, webhooks) still calls them (the deletion checklist, module 16) — a deleted route that a partner's integration depends on is an incident, not a refactor.

## 7. Performance Notes

- **The win to measure (the migration's business case):** the dashboard's *warm* request cost, old vs new: old = 4 fetches + 4 mappings + the poller's 4×30s fetches, *every request*; new = 4 L3 cache hits (~1ms each) in the warm case, 0 poller fetches. The *perceived* win: TTFB (the shell) is now session+org (~10ms) + the fastest section; the sections stream. **The number that matters: warm p95 TTFB** — it should drop from "sum of 4 serialized-ish queries" to "a handful of cache hits" (module 18-01's measurement, before/after).
- **The cold case gets *more* expensive** (honestly): a cold entry re-runs the *service* (the full query + mapping), not just a fetch — the old model's cold case was "re-fetch the URL" (the server-side work was *elsewhere*, in the BFF); the new one's cold case is "run this function here." One indexed query per section — typically a wash or better (the BFF's work moved *into* the entry and is now *cached*; the old model paid it on the poller's *every 30s*). The net: **warm is dramatically cheaper, cold is roughly equal, and the poller is gone** — the sum is a clear win, and the measurement proves it.
- Build time: `cacheComponents` apps prerender the *shells* at build (module 24) — the build does more *rendering* work (the public shell) — on a large site, watch build duration; the head/tail discipline (module 07) is the control.

## 8. Exercise

**Beginner.** Write the **before/after diff** for one legacy function (the revenue section, §4) as a markdown table: old code → new code, one line at a time, with the *meaning change* for each line (response→entry, secret life→profile, inline mapping→DTO, …). This is the translation table's practice — fluency in it is the skill the migration pays for.

**Intermediate.** Build the **legacy fixture**: a minimal Next 14-style app (or a branch of your scaffold with `cacheComponents` off, *simulating* the old model: `fetch`-with-`next.revalidate` style in a plain async function, a polling client island, a BFF route that duplicates the service) — capture its request log (module 19's tracer: per request, what ran). Then run the 6-step migration *for real* on it. Capture the post-migration log. **Diff the two logs**: what stopped running (the poller, the duplicated BFF), what now runs once per entry (the services), what's new (the tags in the log). The diff *is* the migration's verification.

**Production.** The **migration review doc** (`docs/migration-review.md`): for a real legacy codebase (a team's staging app, or a deliberately-built legacy fixture), run Steps 0–6 and document: the Step-3 error list (categorized: key violations vs serialization vs runtime-API), the Step-4 poller deletion (what UX changed, what the teams said), the Step-5 route deletions (the third-party check), the Step-6 audit results, and the before/after p95 TTFB (warm/cold). The doc is the artifact a reviewer signs — the "migration complete" claim, evidenced.

## 9. Architecture Challenge

**Prompt:** A legacy app (Next 14, previous model) has: (1) a public marketing site (static, `revalidate: 3600` fetches, tag `'pages'`); (2) an authed dashboard (fully dynamic — `cookies()` at the top — 4 fetches `revalidate: 60`, a poller); (3) a partner-facing REST API (the BFF routes, `revalidate: 0` — the partners demand "fresh within a second"); (4) a webhook handler that `revalidateTag('orders')`s and `revalidatePath('/dashboard')`s.

The team wants the Cache Components model but is *worried about the partner API* (freshness contract). Design the migration's *target state* for all four surfaces (per surface: what becomes what — entries/profiles/tags vs request-scoped vs plain fetch), and answer the hard question: **can the partner API's "fresh within a second" contract be met in the new model** — and if so, with what (a `seconds` profile? an uncached read? a `remote` cache with sub-second revalidate? a direct DB read from the route?) — and what it *costs* (the module-22 procedure, applied to a contract, not content).

<details>
<summary>Model answer</summary>
Target state:
1. Marketing: entries — `cacheLife('hours')` (the 3600, made visible), `cacheTag('pages')` — the *public* shell (module 24's SHAPE 1): prerendered at build, CDN-served. The biggest, cleanest win (the old model already mostly did this; the migration makes the lives/tags *documents* and the shell *honest*).
2. Dashboard: the module-25 document — per-section entries (`minutes`/`hours`), session request-scoped, poller deleted, the BFF dashboard route deleted (no third party). Straight from this phase's case study.
3. Partner API: the hard one — the "fresh within a second" contract. The honest answer: **don't cache it — read it directly.** The route handler calls the *uncached* service core (`listOrders(orgId, …)` — module 17's service is *already* structured with the uncached core + the cached wrappers: the partner route uses the core, the dashboard uses the wrapper). A `seconds` profile would *work* (a `seconds` entry revalidates in 1s — the module-22 profile) but it's the *wrong tool*: a seconds-life entry is a dynamic hole that *re-runs the service every second under load* (the SWR background check, per entry, per instance) — for a partner API with a *contract*, the direct read is simpler, cheaper to reason about, and *exactly* as fresh (the DB read *is* the freshness). The caching machinery earns its keep where the *change rate is low and the read volume is high* (marketing, dashboards); a high-frequency contract API with per-request reads is the module-24 SHAPE 4 (fully request-scoped) — the *deliberate* uncached path. **The cost:** per-request DB reads at partner traffic (pool sizing, module 21-01's serverless pooling — the reason the contract's traffic volume must be *known* before choosing: 10 req/s is fine direct; 10k req/s needs the cache *and* a contract renegotiation — "fresh within a second" at 10k rps is a different architecture, and the caching model's `seconds` entries + a `remote` (Redis) tier is where you'd land then — but that's a *scale* decision, not a migration decision).
4. Webhook: unchanged mechanics (the Route Handler calls the uncached service core + `revalidateTag` — the module-23 pattern; the `revalidatePath('/dashboard')` **goes** (replaced by the tags — the migration *tightens* the over-broad invalidation), the `revalidateTag('orders')` stays (same primitive, now re-runs the dashboard's *entries* instead of re-fetching the BFF's URLs).
The generalization (the contract question, answered): **caching is a *change-rate* tool; a freshness *contract* is a *SLA* — match the tool to the number: low change rate → entries (any profile); high change rate + high volume → entries with short lives (+ remote tier at scale); high change rate + low volume → direct reads (the contract IS the read). The "fresh within a second" contract at modest volume = direct reads, full stop — and the migration's job is to make the *uncached core* a first-class service (which module 17 already did: the cached wrapper is an *addition* on the core, never a replacement).**
</details>

## 10. Official Documentation

- Caching (the new model, including a "Previous Model" comparison section): https://nextjs.org/docs/app/getting-started/caching
- Caching without Cache Components (the old model, documented for migration): https://nextjs.org/docs/app/getting-started/caching-without-cache-components
- Revalidating (shared primitives): https://nextjs.org/docs/app/getting-started/revalidating
- How Revalidation Works: https://nextjs.org/docs/app/guides/how-revalidation-works

## 11. What You Should Know Before Continuing (Phase 5 complete — the gate)

- [ ] I can state the old model's unit (the fetch response) and the new model's unit (the function entry) — and why the change was made (4 reasons, §1)
- [ ] I can translate any legacy `fetch + next` snippet to a `use cache` function (the table, §2)
- [ ] I know what survives (`revalidateTag`/`revalidatePath`, memoization, tag discipline) and what's gone (the response cache, the route fates, the fetch cache-modes)
- [ ] I can run the 6-step migration (services → directive → key-fix → poller → BFF → audit) and name each step's risk
- [ ] I can answer the contract question (cache vs direct read vs seconds-entry) with a number (volume × change rate)
- [ ] **Phase gate:** the capstone's caching inventory (`docs/caching-inventory.md`) is complete — every surface rowed (WHAT/WHERE/HOW-LONG/WHO/USER-SEES), every mutation spec'd (module 23), the dashboard case study (module 25) written out, the LCP audit done — and the whole thing *is the document a senior reviewer would sign*

**Next phase:** Phase 6 — **Server Functions & Mutations**: the `'use server'` API in depth (naming, serialization, error handling, the mutation path), Server Actions as the mutation standard, and the capstone's mutation inventory. (Phase 5's `updateTag`/`revalidateTag` were the *tails* of the story — Phase 6 is the *head*: the Server Functions they run in.)
