# Module 01 — Why Next.js Exists, and Where Your Code Actually Runs

**Phase 1: Foundations · Module 1 of 101**

> **Where does the code in this module run?** Mostly nowhere — this is the module that teaches you how to label every other module. Code samples are labeled `[SERVER]` / `[CLIENT]`.

---

## 1. Concept — What problem does Next.js solve?

You can already build web apps with React. So what is Next.js for? Five problems, in order of importance:

### Problem 1: A pure client-rendered SPA has no HTML until JavaScript runs

A plain React app (Vite + React) ships a nearly empty `<div id="root">` plus megabytes of JS. The browser must **download → parse → execute** all of it before the user sees a single pixel. Three consequences:

1. **Perceived performance suffers** — First Contentful Paint and LCP depend on JS payload and network.
2. **SEO suffers** — crawlers can index JS-rendered HTML, but with more delay, more fragile indexing, and less predictable serialization of metadata.
3. **Accessibility/fragility suffers** — users with JS disabled or slow connections get a blank page.

**Next.js's answer:** the server renders React to **HTML** and streams it. The browser paints immediately; JavaScript is downloaded in parallel and hydrates (or — with Server Components — some of your UI *never downloads at all*).

### Problem 2: "Server-side rendering" as a bolt-on is the wrong shape

Before the App Router, the common pattern was: React SPA + a *separate* Node/Express backend + a *separate* SSR pass or BFF. That means:

- Two codebases (or one, deployed twice) with two sets of auth, two sets of validation, two sets of types.
- A serialization boundary between "server data" and "client data" you designed by hand.
- Every page load is a roundtrip to your own API that the browser could have avoided.

**Next.js's answer:** the frontend **is** the backend. The same deployment has your UI, your data access, your mutations, and your auth — on one server, with one type system, and a built-in boundary (Server Components ↔ Client Components) instead of an HTTP one you maintain.

### Problem 3: Caching is an architectural decision, not an afterthought

"Make it fast" in a SPA means "add SWR/Query, add Redis, add CDN rules." In Next.js the framework **owns the cache topology**: what is prerendered at build, what is cached per request, what is streamed fresh, and how a mutation invalidates exactly the right entries (tags). You configure a model; the framework enforces it consistently. (This is module 05 in full.)

### Problem 4: The React platform moved, and you need a framework to use it

Server Components, Actions, the `use` API, View Transitions — these are **React** features, but they only work when *something* renders on a server and serializes across the boundary. That "something" is a framework. Next.js is the reference implementation and the production-battle-tested one. React's docs literally point to Next.js as "the easiest way to get started" with RSC.

### Problem 5: The boring stuff (routing, images, fonts, metadata) is a solved problem

File-based routing, `next/image` (AVIF/WebP + responsive + lazy + priority), `next/font` (self-hosting + zero CLS), the metadata API, sitemaps, error/loading/not-found boundaries — shipping all of these correctly by hand is weeks of work with a high defect rate. Next.js ships them as conventions.

**What Next.js is NOT:** a CMS, a hosting provider (it's deployable anywhere), or a required part of React. If you need a marketing site with zero interactivity, plain HTML might be better. If you need a pure admin SPA with a huge existing API, a plain React SPA + TanStack Query might be better. The course will keep saying *when NOT to use it* — that judgment is the actual skill.

---

## 2. Mental Model — the four "where/when" questions

For **every** piece of code you write in this course, answer:

1. **WHERE does it execute?**
   - `[SERVER]` — Node.js runtime on your infra (Vercel, your Docker host, your VPS). Has `process.env`, filesystem, DB, network access to your LAN. The browser never sees this code.
   - `[CLIENT]` — the browser, after hydration. Has `window`, `document`, `localStorage`, GPU, cameras. Never sees your secrets.
   - `[BOTH — BOUNDARY]` — data crossing the seam: serializable props (server→client), action references (client→server), the RSC payload itself.

2. **WHEN does it execute?**
   - **Build** (`next build`) — static output, `generateStaticParams`, font/asset compilation, the **static shell** prerender.
   - **Request** (server) — dynamic holes, runtime APIs (`cookies()`, `searchParams`), Server Functions, Route Handlers, `proxy.ts`.
   - **Client hydration** — `useEffect`, event listener attachment.
   - **Client navigation** — subsequent `<Link>` clicks: RSC payload fetch, no reload.

3. **WHO can read it?**
   - Server secrets: `[SERVER]` code only.
   - `NEXT_PUBLIC_*`: **inlined into the client bundle — public by definition.**
   - Database rows: services only; the UI sees DTOs.

4. **WHAT happens if I get it wrong?**
   - Secret in a client file → **shipped to every visitor in the JS bundle** (grep the production JS to prove it to yourself — exercise below).
   - DB import reachable from a client file → build error (if you use the `server-only` guard) or, worse, a broken runtime crash; the guard makes the failure *loud at build time*.
   - `window` access in a server component → `window is not defined` at render time.

### The server-first default

```
Is a new component a Server Component?  → YES, by default.
Does it need browser APIs / UI state / event handlers? → flip to 'use client'.
```

The **server boundary is a surface area you maximize**: every component that stays on the server is code that never ships, never hydrates, never re-renders client-side, and can touch the database directly. "Keep the server boundary large" is the single most repeated instruction in this course.

---

## 3. Architecture — the request path

```mermaid
flowchart TD
    U([User]) -->|GET /pricing| CDN[CDN / edge]
    CDN -->|miss| N[Next.js server]
    N --> P["proxy.ts [SERVER]"]
    P --> R["Route match: (marketing)/pricing/page.tsx [SERVER]"]
    R --> C1["'use cache' data (cacheLife 'hours') → cache hit [SERVER]"]
    R --> C2["<Suspense> dynamic hole: uncached data [SERVER]"]
    C2 --> DB[(Postgres)]
    R -->|stream HTML + RSC payload| U
    U -->|hydrates Client Components only [CLIENT]| U
    U -->|hover <Link> /dashboard → prefetch RSC payload| N
```

**Five execution surfaces** (the canonical list — you will see it in every architecture review):

| Surface | File convention | Runtime | Can touch DB? | Shipped to browser? |
|---|---|---|---|---|
| Proxy | `proxy.ts` | Node.js (Next 16) | No (shouldn't) | No |
| Server Components | `page.tsx`, `layout.tsx` (no `"use client"`) | Node.js | **Yes (via services)** | No (HTML + RSC payload only) |
| Server Functions | `'use server'` functions/files | Node.js | **Yes** | No (opaque action ref) |
| Route Handlers | `route.ts` | Node.js (or Edge) | **Yes** | No |
| Client Components | `"use client"` files | **Browser** | **Never** | Yes (JS chunks) |

---

## 4. Examples — the smallest possible proof

### 4.1 A page that proves the split

`FILE: src/app/page.tsx` (simplified example — no `src/` yet; the scaffold module moves this)

```tsx
import { ClientClock } from '@/components/client-clock'

// [SERVER] — this component's code NEVER ships to the browser.
// `process.env` and the filesystem are available here.
const serverRenderedAt = new Date().toISOString()

export default function Home() {
  return (
    <main>
      <h1>Server Component page</h1>
      <p>Rendered on the server at: {serverRenderedAt}</p>
      <p>This line is HTML in the initial payload. The browser painted it before any JS ran.</p>
      <ClientClock />
    </main>
  )
}
```

`FILE: src/components/client-clock.tsx`

```tsx
'use client' // [CLIENT — BOUNDARY] everything below this line ships to the browser

import { useEffect, useState } from 'react'

export function ClientClock() {
  const [now, setNow] = useState<string>('')

  // Browser-only APIs: setTimeout, Date on the user's machine.
  // None of this runs on the server during SSR.
  useEffect(() => {
    const id = setInterval(() => setNow(new Date().toLocaleTimeString()), 1000)
    return () => clearInterval(id)
  }, [])

  return <p>Client clock: {now || '(hydrating…)'}</p>
}
```

**What to observe:**
- View source of `/`: you see `Rendered on the server at: …` **in the HTML**. The `ClientClock` markup is there as static HTML (its initial render), but the *behavior* (the interval) starts only after hydration.
- Open DevTools → Sources: there is a JS chunk containing `client-clock` — and **no chunk containing the `Home` component's code or `serverRenderedAt` logic**. That is the entire value proposition of Server Components in one screenshot.

### 4.2 Prove the secret leak (do this — it re-frames the security module)

`FILE: src/app/debug/page.tsx` (BAD — demonstration only)

```tsx
// [SERVER] page
import { SecretBox } from '@/components/secret-box'

export default function DebugPage() {
  return (
    <>
      {/* Fine: secret stays on the server, renders into HTML once */}
      <p>Server sees: {process.env.SUPER_SECRET?.slice(0, 3)}…</p>
      {/* BAD: passing it across the boundary into client JS */}
      <SecretBox value={process.env.SUPER_SECRET} />
    </>
  )
}
```

`FILE: src/components/secret-box.tsx` (BAD — demonstration only)

```tsx
'use client'

export function SecretBox({ value }: { value: string }) {
  return <p>Client sees: {value}</p>
}
```

The secret is now **a string literal in a public JS chunk**. `grep -r "THE-SECRET" .next/static/chunks/` finds it. There is no "hiding" after that — rotate the secret. (Lesson for module 19: the boundary is the security boundary. `NEXT_PUBLIC_` vars are public by design; everything else is server-side or nothing.)

### 4.3 The production pattern: server reads, client interacts

```
[SaaS product — the shape you will build]

page.tsx                          [SERVER]
├── const products = await listProducts({ orgId })   ← service call, direct to DB
├── <ProductFilters />            [CLIENT] — reads/filters via URL, no data fetch
└── <ProductTable data={products} onQuickAdd={action}> [CLIENT] — receives plain data DTOs
        └── <AddProductDialog />  [CLIENT] — calls the 'use server' action
```

The client island receives **data + an action reference**. It never receives the DB, the service, or the org's raw rows beyond what it renders.

---

## 5. Production Code — the "execution label" audit script

A genuinely useful production tool: a script that greps your codebase for boundary violations. Use it in CI from day one (expanded in modules 13 and 19).

`FILE: scripts/audit-boundary.mjs` (production pattern)

```js
// Scans for the classic boundary mistakes and fails CI.
// 1. 'use client' files importing from server-only modules (db, env secrets)
// 2. 'NEXT_PUBLIC_' not used for any env var referenced in a client component (heuristic)
import { readdir, readFile } from 'node:fs/promises'
import { join } from 'node:path'
import process from 'node:process'

const SRC = join(process.cwd(), 'src')
const SERVER_ONLY_IMPORT = /from\s+['"](@\/(lib\/db|db\/)[^'"]*|server-only)['"]/
const CLIENT_FILE = /'use client'/

let violations = 0

async function* walk(dir) {
  for (const entry of await readdir(dir, { withFileTypes: true })) {
    const p = join(dir, entry.name)
    if (entry.isDirectory()) yield* walk(p)
    else if (/\.(ts|tsx|js|jsx)$/.test(entry.name)) yield p
  }
}

for await (const file of walk(SRC)) {
  const src = await readFile(file, 'utf8')
  const isClient = CLIENT_FILE.test(src)
  if (isClient && SERVER_ONLY_IMPORT.test(src)) {
    console.error(`BOUNDARY VIOLATION: ${file} is a client component importing server-only code`)
    violations++
  }
}

if (violations > 0) {
  console.error(`\n${violations} boundary violation(s). Fix before shipping.`)
  process.exit(1)
}
console.log('Boundary audit: clean.')
```

(Real projects also add the `server-only` npm package — `import 'server-only'` in your `db/index.ts` — so the *compiler* rejects client-side imports. Module 09-01 covers this.)

---

## 6. Common Mistakes

| Mistake | Why it's wrong | Fix |
|---|---|---|
| "Next.js is just SSR" | SSR is one of several rendering modes; the framework is the full-stack runtime + cache topology + RSC boundary | Learn the four "where/when" questions; module 05 |
| Whole page marked `"use client"` | You ship the entire page as JS, lose server data access, and re-render everything client-side | Server page + client islands (module 03-04) |
| Server Component using `window` / `localStorage` / event handlers | Those don't exist in the Node runtime; the render crashes or the code silently no-ops | Move that logic to a client island |
| Client Component fetching from your own Route Handler | Extra roundtrip + a serialization seam you maintain | Fetch in the Server Component via services (module 04-01) |
| "I'll pass the whole DB row / object with methods to the client" | Only serializable values cross; functions and Symbols throw at the boundary | DTOs (module 03-03) |
| Passing secrets or assuming `process.env` works the same on both sides | Client bundles inlined at build; `process.env` on the client only contains `NEXT_PUBLIC_*` | Env discipline (module 19-03) |
| Confusing "renders on the server" with "runs once" | A dynamic route renders **per request**; a cached one renders per cache lifetime | The timing model (module 05-05) |

## 7. Security Notes

- **The client bundle is public.** Anything imported by a `"use client"` file, any `NEXT_PUBLIC_*` value, and any prop passed across the boundary is visible to every visitor. Build a habit: treat the bundle as a hostile environment.
- **The proxy is not an auth boundary** (repeated throughout the course; official docs are explicit). It is a network-boundary tool (rewrites, redirects, headers).
- **Secrets have exactly one home: the server runtime.** From day one, run the boundary audit in CI.

## 8. Performance Notes

- **Server Components reduce your client JS** — the most structural performance win in modern React. Before micro-optimizing, count how many components you *don't* ship.
- **Time to First Byte (TTFB) and LCP** are now server responsibilities: a slow database query in a Server Component is a slow first paint. (Module 18 measures this.)
- **Hydration cost** is proportional to client component count. Every `"use client"` is a budget item.

## 9. Exercise

**Beginner.** Build the 4.1 example in a scratch project. Then: (a) view source and point to the server-rendered string in the HTML; (b) find the `client-clock` chunk in the browser sources; (c) confirm there is **no** chunk containing the `Home` page component. Screenshot all three.

**Intermediate.** Build a page with one Server Component parent and three client children: a counter, a modal, and a theme toggle. For each file, write a one-line label comment `[SERVER]`/`[CLIENT]` explaining *why* that label (which of the 5 decision steps fired). Then deliberately break one: move the counter's state into the server component and read the exact error message. Paste the error and explain it in your own words.

**Production.** Add `scripts/audit-boundary.mjs` to your capstone scaffold and wire it into CI (a GitHub Actions step). Create a *deliberate* violation in a branch, confirm the CI fails with a clear message, then fix it. Bonus: add the `server-only` package to your (future) `db/` module and verify the compiler error when a client file imports it.

## 10. Architecture Challenge

**Prompt:** You join a team whose "Next.js" app is: every page `"use client"`, data fetched in `useEffect` from their own `/api/*` route handlers, session JWT stored in `localStorage`, and an Express backend duplicating the API routes.

Before reading the model answer, reason out — in writing, in this order:
1. Which of the four "where/when" questions is violated by each of those four choices?
2. What can an attacker see/read with just DevTools open, that they shouldn't?
3. What does the first request to a product page look like on the wire (list the hops)?
4. If you could only fix **one** thing first, which one, and why?

<details>
<summary>Model answer (reason first)</summary>

1. `"use client"` pages: WHERE violated — code that could run server-side runs in the browser, so WHEN (no build-time prerender of those pages) and WHO (JS shipped to everyone) follow. `useEffect` fetching: WHERE + WHEN violated — data loads *after* paint (LCP suffers) and the fetch is a redundant hop through your own API. `localStorage` JWT: WHO violated — any XSS can read the token; it is also sent by your code (CORS-dependent), not automatically by the browser. Express duplicate: violates the single-type-system/single-deployment premise — two auth implementations to keep in sync.
2. The JWT (and anything else in `localStorage`), all API responses, and — because pages are client components — the full client bundle including any embedded `NEXT_PUBLIC_*` and sometimes *logic* that reveals internal endpoints/schemas.
3. HTML shell (empty) → JS bundle(s) → parse/execute → hydration → `useEffect` fires → `GET /api/products` (your API) → (often) your backend DB. Four network hops minimum before first content from data.
4. **The `localStorage` JWT** — it is a security incident waiting to happen, and it also couples every client fetch to CORS + token-sending code. Replacing it with an HttpOnly cookie session changes the security model, simplifies the client fetches, and (in Next.js) lets the proxy/Server Components read the session without any client code. Then refactor pages server-first, which eliminates the self-API hops.
</details>

## 11. Official Documentation

- Next.js docs home & version: https://nextjs.org/docs
- Getting started: https://nextjs.org/docs/app/getting-started
- Server and Client Components: https://nextjs.org/docs/app/getting-started/server-and-client-components
- Fetching data: https://nextjs.org/docs/app/getting-started/fetching-data
- Mutating data (Server Functions/Actions): https://nextjs.org/docs/app/getting-started/mutating-data
- Proxy: https://nextjs.org/docs/app/getting-started/proxy
- Data Security guide: https://nextjs.org/docs/app/guides/data-security
- React — Server Components: https://react.dev/reference/rsc
- React — "Why use Server Components": https://react.dev/learn/components-on-the-server
- What's new in Next.js 16: https://nextjs.org/blog/next-16

## 12. What You Should Know Before Continuing

- [ ] I can explain, in one sentence each, the five problems Next.js solves
- [ ] I can label any file `[SERVER]`/`[CLIENT]` using the 5-step decision procedure, and say what breaks if I'm wrong
- [ ] I can list the five execution surfaces and which runtime each uses
- [ ] I understand that `NEXT_PUBLIC_*` is public by definition, and I've seen a secret leak in the production bundle
- [ ] I know "static is faster" is incomplete — speed is a function of *what* is served from *which* cache layer *when*
- [ ] I have the boundary audit running in CI

**Next:** Module 02 — Project Setup: `create-next-app` deep dive (every option explained, and we turn on Cache Components).
