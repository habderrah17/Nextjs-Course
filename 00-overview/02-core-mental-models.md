# 02 — Core Mental Models

> Read this file **before any API in the course.** Everything else is elaboration of what is established here. You will return to it after every module; that is not repetition, it is the point.

---

## Model 1 — Where code executes (the Server/Client boundary)

The single most important fact about Next.js: **the same React file can contain code that runs only on the server, code that ships to the browser, and code that does both — and the unit of that distinction is the component.**

```
Browser (JavaScript runtime: the user's machine)
   │
   ▼  HTTP (what crosses the wire: HTML, RSC payload, JS chunks, cookies, POST bodies)
Next.js Server (Node.js runtime: YOUR machine / Vercel / your infra)
   │
   ├── proxy.ts                      [SERVER — runs on every matching request, before routing]
   ├── Server Components             [SERVER — render to HTML/RSC, never ship as JS]
   │     ├── services → Drizzle      [SERVER — database access]
   │     └── outbound fetch          [SERVER — external APIs, server-to-server]
   ├── Server Functions ('use server')[SERVER — invoked by POST from the client]
   ├── Route Handlers (route.ts)     [SERVER — raw HTTP endpoints]
   │
   └──► ships to browser:
        Client Components ('use client')  [CLIENT — hydrated JS with state & events]
```

### The three labels you will use forever

| Label | Meaning | Examples |
|---|---|---|
| `[SERVER]` | Executes only in the Node.js runtime. Has access to `process.env` secrets, filesystem, database, `cookies()`, `headers()`. **Never** reaches the browser. | `layout.tsx` (RSC), services, Drizzle client, `proxy.ts`, `route.ts`, Server Functions |
| `[CLIENT]` | Shipped to the browser as JavaScript, hydrated there. Has `window`, `document`, `localStorage`, event handlers, browser timers. **Never** sees server secrets. | `use client` components, their hooks, Motion components, RHF forms |
| `[BOTH — BOUNDARY]` | The seam where data crosses. Props from server→client are **serialized**; actions cross as opaque references; what can cross is a defined, finite set. | `<ClientIsland data={…} onSave={action} />` |

### Decision procedure (run for every component you write)

```
1. Does it need browser APIs (window, clipboard, IntersectionObserver, canvas, pointer events)? → CLIENT
2. Does it hold per-user UI state (open/closed, drag position, animation frame)? → CLIENT
3. Does it render data the user can't change without a roundtrip? → SERVER (default)
4. Does it fetch its own data? → SERVER (fetch in the server, pass data down)
5. Otherwise → SERVER
```

If the answer is SERVER, the component can: query the database directly, read cookies/headers, import secret env vars, and **its JSX is generated on the server and streamed as HTML** — the browser never downloads the component's code.

### What "use client" actually means

`"use client"` at the top of a file is a **boundary marker, not a runtime switch**. It tells the compiler: *this file and everything it imports are client components; they must be shipped to the browser and hydrated.* The module is then:

1. Bundled as a client JS chunk,
2. Loaded on the page,
3. **Hydrated** — React attaches event listeners and state to the server-rendered HTML,
4. **Invokable** as props: a server component can render `<ClientIsland>` and pass it serializable props.

Everything *outside* the boundary (the importing server component's code) still runs on the server. `"use client"` is a ratchet: code can import client→up, but the moment you cross into a client file, everything below it is client.

---

## Model 2 — When code executes (the timing model)

Next.js runs code at **four different times**. Confusing these causes most "why is my data stale/missing" bugs.

| Time | What runs | Example |
|---|---|---|
| **Build time** (`next build`) | Static rendering of pages that don't need request data; `generateStaticParams` values; font compilation; asset optimization. Output: static HTML + cached RSC payloads + the **static shell** (with Cache Components: everything within a `cacheLife` that's long enough). | Landing page, product pages for known IDs |
| **Request time (server)** | Dynamic holes in the shell: components reading `cookies()`/`headers()`/`searchParams`, uncached data fetches inside `<Suspense>`, Server Functions, Route Handlers, proxy. | "Welcome back, Dana" greeting, user dashboard |
| **Client time (hydration)** | Client components initialize state/effects. Your `useEffect` code. | Opening a modal, playing a sound |
| **Client time (post-hydration navigation)** | The client router asks the server for the next route's RSC payload (no full reload), React reconciles; prefetching may have done this ahead of time. | Clicking `<Link>` in an authenticated dashboard |

**The mental picture of a request** (App Router, Cache Components on):

```
1. proxy.ts runs (Node.js) — can rewrite/redirect/respond
2. Router matches the route → layout tree
3. Server Components render:
     • cached (use cache, within life)  → served from cache, no work
     • uncached async inside <Suspense> → fallback streamed first, data streams in
     • runtime APIs (cookies…)          → dynamic hole, request-time work
4. HTML + RSC payload stream to the browser (progressive)
5. Browser paints the shell immediately; dynamic holes fill as they arrive
6. Client components hydrate (priority: forms → interactive → rest)
```

---

## Model 3 — The data model (who owns data, where it flows)

```mermaid
flowchart LR
    subgraph CLIENT
        UI[Client Component UI]
    end
    subgraph SERVER
        SC[Server Component]
        SVC[Service layer]
        SF[Server Function]
        RH[Route Handler]
        CACHE[(Next.js data cache<br/>Cache Components)]
    end
    subgraph INFRA
        DB[(PostgreSQL)]
        EXT[External APIs]
    end

    SC --> SVC
    SF --> SVC
    RH --> SVC
    SVC --> CACHE
    SVC --> DB
    SVC --> EXT
    UI -->|form POST / action ref| SF
    UI -->|fetch / query client| RH
    UI <-->|props (serializable)| SC
```

Rules:

1. **Server components read data directly** (server → service → DB). There is no API hop between a server component and the database. An internal API route between your own server code is **almost always wrong** (it adds latency, a serialization boundary you maintain, and a second auth check for nothing).
2. **The client never talks to the database.** Client → (a) Server Function for mutations/forms, or (b) Route Handler for fetch-style reads (rare in your own app; see the decision matrix in module 07-05).
3. **Services are the only code that knows about the ORM.** Server Components and Server Functions call services; services return **DTOs** (plain serializable objects) — never ORM rows with relations/`Date`-heavy internal state beyond what the UI needs.
4. **Tenancy is applied at the service boundary** (module 11): `listProducts({ orgId })` — `orgId` comes from the session, never the URL/body.

**BAD architecture** (this is how tutorials teach it — don't):
```
Client Component → fetch('/api/products') → your own Route Handler → DB
```
when the page itself is a Server Component. The component should just `const products = await listProducts()`.

**WHEN a Route Handler is right** (module 08): external consumers (mobile app, partner API), webhooks (Stripe), file downloads/uploads, streaming responses (SSE), or a BFF layer when real backend services exist behind you.

---

## Model 4 — The caching model (six layers, one question)

Data can sit in **six different caches** in a Next.js stack. When anything is "slow" or "stale," first name the layer:

| # | Layer | Where | What lives there | Who controls lifetime |
|---|---|---|---|---|
| 1 | **Browser HTTP cache** | user's machine | HTML, JS chunks, images (per HTTP headers) | `Cache-Control`/`ETag` headers |
| 2 | **CDN / edge cache** | network edge | Static assets, static HTML shell (RSC cache at the edge on supported platforms) | platform + `cacheTags`/profiles |
| 3 | **Next.js data cache (Cache Components)** | server process memory (default) or your `cacheHandlers` (Redis etc.) | return values of `'use cache'` functions/components, keyed by build ID + function ID + serialized args | `cacheLife` (stale/revalidate/expire) + tags |
| 4 | **Request memoization** | one request only | same uncached fetch/call within a render pass runs once per request (per key) | request scope |
| 5 | **Client cache** | browser JS memory (if you add one, e.g. TanStack Query) | client-fetched server state | your library config |
| 6 | **Database / cache layer** | infra | DB query cache, connection pool, PgBouncer, CDN behind external APIs | infra config |

**The five questions for every cached thing** (answer in writing; this is an assignment format):

1. **WHAT** is cached? (a data value / a UI fragment / a whole page / a session?)
2. **WHERE** does it live? (layer 1–6)
3. **FOR HOW LONG**? (`cacheLife` profile, or time-based vs. event-based)
4. **WHO invalidates it**? (a mutation's `updateTag`/`revalidateTag`, a time expiring, a deploy, a user action)
5. **WHAT does the user see after invalidation**? (stale-while-revalidate: stale now, fresh in background / hard wait: block until fresh / immediate: read-your-own-writes via `updateTag`)

And the **ownership rule**: cache keys must include *everything* the data depends on. `getProducts(orgId, category)` cached without `orgId` in its key = cross-tenant data leak (layer-3 bug). This is why runtime data (cookies/headers) is read *outside* cached scopes and passed in as arguments — arguments are automatically part of the cache key.

---

## Model 5 — The authentication/authorization model (two gates, three layers)

```mermaid
flowchart TD
    REQ([Request]) --> P{proxy.ts<br/>[SERVER — optimistic]}
    P -->|no session cookie| R1[redirect /login<br/>(UX shortcut only!)]
    P -->|cookie present| L{Layout / Server Component<br/>[SERVER — THE BOUNDARY]}
    L -->|getSession: no session| R2[401 → /login<br/>(authoritative)]
    L -->|session valid| A{Authorization check<br/>[SERVER — service layer]}
    A -->|no permission| R3[403 or notFound<br/>(authoritative)]
    A -->|allowed| UI([Render / mutate])
```

- **Gate 1 — Authentication (ARE YOU WHO YOU SAY?):** cookie → session row in your DB → user. Done in the layout/server component, authoritatively. The proxy's redirect is a *UX optimization* (skip a render, don't serve a login page to someone who obviously isn't logged in) — official docs are explicit it is **not** a security solution.
- **Gate 2 — Authorization (MAY YOU TOUCH THIS?):** role/permission + **resource ownership** + **tenant scope**, enforced in the service layer on *every read and every mutation*. Frontend hiding of buttons is presentation, not security — a curl request reaches the same Server Function.
- **Three layers, distinct jobs:** UI (hides/affords), server (enforces), database (last line: constraints, and RLS if you want defense in depth).
- **401 = unknown identity (gate 1 failed). 403 = known identity, no permission (gate 2 failed).** Not-found is often the better 403 for existence-orbital resources (don't leak that a resource exists).

---

## Model 6 — The deployment model (the same code, different physics)

The physics of your deployment change five things. Every "it just works" claim in this course has a deployment footnote:

| Physics | Long-lived Node server (Docker/VPS) | Serverless (Vercel & kin) | Edge runtime |
|---|---|---|---|
| **Process lifetime** | minutes–months | per-invocation, cold/warm | per-invocation, very fast cold start |
| **Filesystem** | persistent | **ephemeral** → no app-filesystem storage in prod | none |
| **DB connections** | direct pool OK | **pooler required** (PgBouncer/managed) | HTTP-based drivers only |
| **`use cache` in-memory store** | survives between requests (one process) | per-instance, may not survive → durable via `cacheHandlers` / `'use cache: remote'` | minimal |
| **Long-running work** | in-process workers fine | queues + external workers (time limits) | not for long work |
| **Env vars** | `.env`/secret manager | platform | platform (subset) |

**Mental rule:** write code that assumes *any* of the three (no temp-file writes, no module-level singletons that hold DB state, no long loops in requests) and you can deploy anywhere.

---

## The five models, in one paragraph

A request hits your **proxy** (server, network edge of *your* app); **Server Components** render on the server, reading from **six cache layers** and the database through **services** (which enforce **tenant scoping**); **runtime data** (cookies/search params) and **uncached async work** become dynamic holes streamed behind Suspense fallbacks; the browser paints the shell, hydrates **Client Components** (the only JS you shipped), and afterwards navigations are RSC-payload updates instead of reloads; **Server Functions** handle mutations over POST with validation, auth, and tag-based **cache invalidation**; and all of it runs on **deployment physics** you chose deliberately. Everything else in this course is detail.

---

## Exercise (do this before Module 1)

1. Open the dev server of any Next.js 16 app you can build (module 01-02 walks you through it). For the default `app/page.tsx`, write down: which of the six cache layers hold its output, at which of the four execution times its code runs, and which label ([SERVER]/[CLIENT]) applies to every file in the default template.
2. Take one React app you built in your React course (SPA, all client). List the **server** it doesn't have: where would `process.env` secrets live? Where would the session live? Where would search/filter server-side? Write three sentences on what breaks in the SPA model at 10× the data. (This is the argument for Next.js; module 01-01 formalizes it.)

## What You Should Know Before Continuing

- [ ] I can state the three labels and apply the 5-step decision procedure to any component
- [ ] I can name the four execution times and say which one applies to: `next build`, a dashboard request, a `useEffect`, a link click
- [ ] I can list the six cache layers and answer the five caching questions for a given piece of data
- [ ] I can explain why the proxy redirect is UX, and where the *real* auth check happens
- [ ] I can explain why server component → own API route → DB is a smell
- [ ] I can state what changes about caching/storage/connections when moving Docker → serverless → edge
