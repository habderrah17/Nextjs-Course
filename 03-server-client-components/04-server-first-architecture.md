# Module 15 — Server-First Architecture: BAD vs GOOD, and When SPA-Style Is Right

**Phase 3: Server/Client Components · Module 15 of 101 (closes Phase 3 → Architecture Review #1)**

> **Where does this run?** This module is the *policy* that the previous three established mechanically: default server, islands on the client, and a small, explicit set of exceptions.

---

## 1. Concept — Server-first is a *rendering* policy with economic reasons

"Default to Server Components" is not ideology. It is the cheapest correct default for the five costs every page pays:

| Cost | Server-first | SPA-style (all client) |
|---|---|---|
| First paint | HTML streams immediately | JS must download+execute first |
| Client JS shipped | Only islands | The whole page |
| Data path | Service → DB (1 hop) | Client → API → (maybe) DB (2+ hops) |
| Cacheability | Static shell + tagged data | Nothing at the edge (request data) |
| Auth surface | Session read in one place | Token/cookie logic in client code |

Server-first means: **the component tree is server-rendered and data-bearing by default; `"use client"` is a *promotion* you justify per component.** The promotion criteria (from the island contract, module 13): browser APIs, interaction state, event handlers, client-only libraries, animation/gesture code.

## 2. Mental Model — the "surface area" budget

Think of client JS as a budget you *spend*, not a default you *inherit*:

```
route: /products/manage
  client islands: Toolbar (search draft, filters)      ← earned: interaction
                 ProductTable (row actions, selection) ← earned: interaction
                 PaginationLinks (prefetch targets)    ← earned: navigation
                 AddProductDialog (form + action)      ← earned: form state
  server: everything else (data, structure, chrome)
  spend: ~60KB route-scoped
```

The audit question for every route (and every architecture review): **"For each client island on this route, name the interactivity it owns."** If you can't, the island is a liability — convert it (or delete it).

## 3. Architecture — the transformation patterns

### Pattern A: The "use client page" → server page + islands (the big one)

BAD (module 13 §4.3, condensed): the page fetches in `useEffect`, renders a spinner, everything is client.

GOOD: server page reads session + fetches DTOs + composes islands. **The transformation checklist:**

1. Move data reads to the page (server), via services.
2. Split each interactive widget into a client component with a *narrow* props interface (DTOs + actions).
3. Move pending/error/empty *rendering* to the server (it has the data — it knows if it's empty); move pending/error *state* for client-side mutations to the island (`useFormStatus`, `useOptimistic` — module 07-04).
4. Delete the client's `fetch` calls and their `useEffect` waterfalls.
5. Verify: HTML in View Source is complete; islands are the only JS; network shows no `/api/*` self-calls.

### Pattern B: The "context god object" → server props + one store

BAD: a `UserDataContext` provider that fetches the user in `useEffect` and every component consumes it.
GOOD: the **layout** (server) reads the session once, passes what the chrome needs as props; a *thin* client context exists only for values that must update client-side (theme, cart count) — sourced from the server's initial render and a small store (module 23-02).

### Pattern C: The "client data grid" → server data + client chrome

BAD: a grid library doing its own fetching, filtering, and sorting in the browser (a 10k-row JSON blob shipped).
GOOD: the server does search/filter/sort/pagination from the URL (module 10 + 17-01); the grid client component renders the *page* of rows and emits URL changes. The grid library keeps its *interaction* role (column resize, virtualization if needed) but loses its *data* role.

### Pattern D: The "polling client" → cache profiles + refresh

BAD: `setInterval(fetch, 30000)` in a client effect.
GOOD: the section's data has a `cacheLife` matching its real freshness needs (module 05-03); the client uses `router.refresh()` on a longer cadence only where live-ness matters, or SSE for true real-time (module 22-04 area). The *server* owns freshness; the client asks, it doesn't re-implement.

## 4. When SPA-style is actually right (the honest exceptions)

The course is server-first; these are the cases where you *deliberately* go client-heavy, and you should be able to argue them:

1. **Tools with dense, continuous client interaction**: canvas editors, code editors, design tools, games. The "page" *is* the interaction; server rendering adds latency to a loop that runs at 60fps. (Next.js still hosts them fine — as client islands on an otherwise server page.)
2. **Apps with a large existing API and no intent to co-locate the backend**: a React SPA + TanStack Query in front of a 5-year-old API is often the *lower-friction* architecture than migrating the backend. (Module 23-01 architecture #3.)
3. **Offline-first features**: local-first data (IndexedDB/CRDTs) where the server is a sync target, not the source. (Module 23-03's "when Query earns its keep.")
4. **Widget/embed contexts**: a component shipped into *other* sites (no server of your own in the request path) is structurally an SPA.

The decision rule: **the backend you're rendering against must be *reachable at render time, from the server*, and the interactivity must not *be* the product.** If either fails, SPA-style is defensible — say so in the architecture doc, because reviewers will ask.

## 5. Production Code — the reviewed artifact (Architecture Review #1)

After Phases 1–3, the capstone's public + authed shells exist. The review artifact:

`FILE: docs/review-1-boundaries.md` (template — fill it for your app)

```md
# Architecture Review 1 — Server/Client Boundaries
Date: …  App state: Stage 3 (public site + shells)

## Route inventory
| Route | Server components | Client islands (interactivity owned) | Verdict |
|---|---|---|---|
| / | Hero, Features, PricingTable | ThemeToggle (theme), FaqAccordion (open state) | ✅ |
| /products | Page, ProductGrid, Pagination | Toolbar (filters), ProductCard actions (wishlists) | ✅ |
| /products/[slug] | Page, Hero, Gallery | BuyBox (cart add), VariantPicker | ✅ |
| /dashboard | Page, all data sections | OrdersTable (cancel), Filters | ⚠ OrdersTable: review cancel UX (pending state) |
…

## Boundary violations found
- (none / list with fix)

## Client JS budget per route
| Route | Chunks (KB, gzip) | Largest chunk → which island |
|---|---|---|
…

## Decisions
- D1: … (e.g., "BuyBox stays client: form state + optimistic cart update")
```

This template is reused at every review (#2 caching, #3 authz, #4 data screens, #5 deployment) — the course's recurring checkpoint (module 23-04).

## 6. Common Mistakes (the compendium begins here)

| Mistake | Why it persists | The kill |
|---|---|---|
| Whole-page `"use client"` | "It's how my React app works" | Pattern A; the HTML-in-View-Source test |
| Context for data | Context is for *sharing*, not *fetching* | Layout reads once, passes down |
| Client grid with server data re-fetched | Library habit | Pattern C: grid renders the prop'd page |
| `useEffect` fetch "just for this one widget" | The widget was written as a standalone SPA component | The parent server component is its data source |
| Forgetting islands also *serialize* | The island's props are wire bytes | Module 14: narrow DTOs, on-demand detail |
| "We'll refactor after launch" | The refactor is the architecture; deferring it ships the SPA | Refactor per-pattern now; it's hours, not weeks, per page |

## 7. Security Notes

- The boundary is the security boundary (repeated on purpose): every BAD pattern in §3 is also a *security* regression — client fetches mean client-side auth decisions, client state means secrets-adjacent data in inspectable JS, and no server render means no server-side "don't render what the user can't see."
- Review checklist line: "For each route, can a logged-out user see any user-specific data in the *initial HTML*?" (The proxy + layout gates say no — verify it.)

## 8. Performance Notes

- The server-first page's LCP is the *server's* problem (fast queries, cached data — module 05); the SPA page's LCP is the *network + bundle + JS* problem. The former has a much higher ceiling and a much better floor.
- Route-scoped islands keep chunks small; the bundle analyzer (module 18-03) is the verifier.
- React Compiler (module 02-02) removes *re-render* cost from islands — it does **not** remove the cost of *shipping and hydrating* them. Budget is still budget.

## 9. Exercise

**Beginner.** Write the "island interactivity" table for every client component in your scaffold (even if it's three components). For each: one sentence, what it owns, what it receives, what it invokes.

**Intermediate.** Pick the worst page in a real (or your own) legacy app. Execute Pattern A fully: document before/after (HTML in source, chunk list, self-API calls). Time-box it: if the page is under 500 lines, the refactor should be under 2 hours.

**Production.** Run Architecture Review #1 on the capstone using the §5 template. Include the client JS budget table (run `next build`, read the route chunk sizes). Flag any route where a chunk exceeds 50KB gzip and write the one-line plan to cut it.

## 10. Architecture Challenge

**Prompt:** A stakeholder demands: "The dashboard must feel instant — every click should be a full-page refresh with everything re-fetched, so the data is always fresh."

Translate their requirement into the *actual* problem, then design the architecture that satisfies it *without* a full-page refresh per click (you may use at most one full refresh in the flow). Cover: what "always fresh" means for each dashboard section (is it true?), what the caching model does for it, and what you'd tell the stakeholder the tradeoff is.

<details>
<summary>Model answer</summary>
The actual problem: "I make decisions from this dashboard; if the numbers are stale I lose money." Not "I want refreshes."
Design: sections get the cache profiles their real change-rate supports (module 05-06: revenue `hours`, orders `updateTag` after mutations so the *actor* is always fresh, activity `minutes`, analytics `hours`). "Always fresh" is *true* for the thing that matters — the user's own actions (read-your-own-writes via `updateTag`/redirect). For genuine live-ness (e.g., order status), one section uses a 30s `router.refresh()` cadence or SSE — that's the *one* refresh-like behavior, scoped to the one section. Full-page refresh on every click is the *worst* answer: it throws away the streaming shell, re-runs every query at once (waterfall), and still shows stale data for anything with a >0s cache life — freshness comes from *invalidation*, not refresh frequency.
To the stakeholder: "You'll see your own changes instantly (that's the guarantee that matters for decisions). Market-wide numbers refresh on their real schedule — a number that 'changed' every 5 seconds because we re-queried is not a fresh number, it's a noisy one. Here's the one section that's genuinely live, and here's its latency."
</details>

## 11. Official Documentation

- Server and Client Components: https://nextjs.org/docs/app/getting-started/server-and-client-components
- React — RSC: https://react.dev/learn/react-server-components
- Caching: https://nextjs.org/docs/app/getting-started/caching
- Client-side data fetching (when Query is right): https://nextjs.org/docs/app/guides/client-side-data-fetching

## 12. What You Should Know Before Continuing

- [ ] I can state the economic reasons for server-first (the 5-cost table) from memory
- [ ] I can execute Patterns A–D and name a real example of each
- [ ] I can argue the 4 SPA-style exceptions honestly (they're real)
- [ ] I have the §5 review template and know it's reused at reviews #2–#5
- [ ] Architecture Review #1 is done for the capstone

**Phase 3 complete.** **Next:** Phase 4 — Data Fetching. Module 16: Fetching in Server Components (and why your own API route is almost never the answer).
