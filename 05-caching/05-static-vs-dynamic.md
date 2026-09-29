# Module 24 — Static vs Dynamic Rendering: Why, When, and How Caching Changes the Equation

**Phase 5: Caching · Module 24 of 101**

> **Where does this run?** "Static" work happens at **build** (`[SERVER]`, one machine, once). "Dynamic" work happens at **request** (`[SERVER]`, the fleet, per user). The modern model's claim: this is no longer a binary *per page* — it's a *per read* decision, and caching is what moves the line.

---

## 1. Concept — The old dichotomy, dissolved

**The pre-16 mental model** (still correct for "Previous Model" codebases — module 27): a route is either *static* (prerendered HTML, served from cache, revalidated on a timer/path) or *dynamic* (rendered per request, using request data). The choice was per-route, made by what the route *touches* (`cookies()` anywhere in the tree → the whole route goes dynamic), and "static is faster" was the whole argument.

**The Cache Components model** (this course): the route's output is a **static shell** — everything within `use cache` scopes whose `cacheLife` is long enough to store — plus **dynamic holes**: the short-lived/uncached reads and the runtime-API reads, streamed behind Suspense fallbacks at request time. The same URL serves both. The unit of the decision moved from *route* to *read*:

```
OLD:   page touches cookies()?  →  WHOLE page is dynamic (per request, always)
NOW:   page has a cached catalog read (hours) + a session read (request)
       →  shell: the catalog section (CDN-served, prefetchable)
          hole:  "Hello, Dana" (request-time, streamed in 200ms behind a skeleton)
```

**Why "static is faster" is the incomplete sentence** (the user's explicit requirement — here's the full one): static output is faster *to serve* (no server work per request; CDN edge; prefetched before the click), but static output is only *correct* while it's fresh. The full sentence: **static is faster *for the freshness you can afford*** — and with caching, "how much freshness you can afford" is a *per-data* dial (`cacheLife` + tags), not a *per-page* commitment. A page can be 90% static-shell and 10% live, and the 10% costs nothing until someone requests it.

## 2. Mental Model — build time vs request time, precisely

### What happens at `next build` (build time)

1. TypeScript check, bundle compilation (client + server) — Turbopack.
2. **Prerendering**: every route reachable without request data is rendered:
   - static routes (all of `(marketing)`) — full HTML + their cached reads (the shell)
   - `generateStaticParams` values (known slugs)
   - the **static shell** for dynamic routes: everything within storable `cacheLife` renders at build *with the data it can get at build*; the rest becomes holes.
3. Font/asset compilation, metadata generation, output written to `.next/`.

What build-time rendering *cannot* do: read `cookies()`/`headers()`/`searchParams` (they don't exist at build), call a provider that's down at build (that value becomes a hole or a build error — deliberate), or know a user.

### What happens at request time

1. Proxy (Node.js) — matcher, optimistic checks.
2. Route match; layouts render: the **session read** (request-scoped) is the first "real" per-request work on authed routes.
3. The page: cached reads hit (memory/CDN — ~free), holes execute (queries, external fetches) and **stream** behind their fallbacks.
4. The browser paints the shell (which it may have *already* had via prefetch — module 11), then fills holes as they arrive.

**The equation:** `perceived time = max(shell delivery, first hole)` for the first section, and each hole lands *independently*. Caching changes the equation by making shell delivery ≈ CDN hit (50ms) and hole execution ≈ cache hit (1–10ms) *in the common case* — while the *correctness* of each piece is governed by its own life/tag, not the page's fate.

## 3. Architecture — the decision table (the module's artifact)

For **each read** in a page (the inventory's per-section rows), answer four questions:

| Question | Answers |
|---|---|
| **Does it depend on the request?** (user, URL params, time-of-day) | no → cacheable (data/UI level) · yes (runtime API) → read outside + pass as args, or dynamic hole · yes (unavoidable: the user's own state) → request-scoped, never cached |
| **What's its change rate?** | → the `cacheLife` profile (module 22 procedure) |
| **Can it be in the shell?** (do you want it CDN-served/prefetched?) | life long enough + no request dependency → yes · short-lived → hole by design |
| **What's its invalidation?** | tag + owning mutation (module 23 spec) · time-only (external data) · none (`max`/deploy) |

**The four canonical shapes** (memorize; every page is a combination):

1. **Fully static** — landing/pricing/legal: all reads cached long (`days`–`max`), no request data. Shell = whole page. CDN + prefetch. Invalidation: deploy + occasional tags.
2. **Shell + live holes** — the public product detail: catalog data (`hours`, shell) + a live stock badge (`seconds`, hole). The *common* shape for content sites.
3. **Per-user dynamic** — the dashboard: session read (request-scoped) + user-scoped cached reads (keyed by orgId — *cached, but per user*) + a live section (hole). The "dynamic" page that's mostly *cached* — the old model would have called this "dynamic, render per request, always slow"; the new model renders the shell at build, caches per-org data, and streams only the live bits.
4. **Fully request-scoped** — the checkout: every price, every inventory check, the session — *deliberately uncached* (the freshness cost of a stale checkout is an incident). Slower per request; *correctness is the product*.

## 4. Production Code — one page, all four shapes visible

`FILE: src/app/(marketing)/products/[slug]/page.tsx` (the annotated production page — [SERVER])

```tsx
import { notFound } from 'next/navigation'
import { Suspense } from 'react'
import { z } from 'zod'
import { getProductBySlugCached } from '@/services/catalog'     // SHAPE 2: shell (hours-cached)
import { getStockLevel } from '@/services/inventory'             // SHAPE 2: live hole (seconds-cached)
import { listRelatedProducts } from '@/services/catalog'         // SHAPE 1-ish: hours-cached (shell)
import { ProductSkeleton } from '@/components/product-skeleton'

const paramsSchema = z.object({ slug: z.string().regex(/^[a-z0-9-]+$/) })

export async function generateStaticParams() {
  // Head products prerendered at build (module 07): known slugs → full shell at build.
  return (await listPublishedProductSlugs()).map((slug) => ({ slug }))
}

export default async function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params
  if (!paramsSchema.safeParse({ slug }).success) notFound()

  // The catalog read is 'use cache' + 'hours' + tag — this page's SHELL (for known slugs,
  // it was rendered at build; for the tail, it's the first request that fills the entry).
  const product = await getProductBySlugCached(slug)
  if (!product) notFound()

  return (
    <article>
      <ProductHero product={product} />          {/* shell: static, instant, CDN */}

      {/* The live hole: stock changes continuously; 'seconds' profile → excluded from the
          shell by design (module 22: short lives are holes). Streams in ~200ms behind a chip. */}
      <Suspense fallback={<StockChipSkeleton />}>
        <StockLevel slug={slug} />
      </Suspense>

      {/* Related: hours-cached data → shell-eligible; still wrapped so a cold tail
          entry doesn't block the hero. */}
      <Suspense fallback={<RelatedSkeleton />}>
        <RelatedProducts slug={slug} />
      </Suspense>
    </article>
  )
}

async function StockLevel({ slug }: { slug: string }) {
  const level = await getStockLevel(slug)        // 'use cache' · cacheLife('seconds') · no tag (time-based)
  return <StockChip level={level} />
}

async function RelatedProducts({ slug }: { slug: string }) {
  const related = await listRelatedProducts(slug) // 'use cache' · 'hours' · 'catalog' tag
  return <RelatedGrid items={related} />
}
```

**Read the page as a caching document:** the shell (hero + related, hours/days-cached) is buildable and CDN-able; the stock chip is *deliberately* a hole; the page 404s at the boundary (slug schema) and at the read (unknown slug) — no empty shells for non-products.

## 5. Common Mistakes

| Mistake | The model it violates | Fix |
|---|---|---|
| "It reads `searchParams`, so the whole page is dynamic" | The old per-route dichotomy | Parse params in the uncached scope; feed *cached* functions with the parsed values as args — the *data* can still be cached per-param-combination (module 21's key = args) |
| Forcing a page dynamic to "be safe" (`force-dynamic` habit from the old model) | You've paid per-request rendering for a page that's 95% stable | The per-read decision table; audit which read *actually* needs request-ness |
| Expecting `generateStaticParams` to cover an infinite space | Build bloat (module 07) | Head at build, tail as holes/shell |
| Caching a request-dependent read *without* the dependency in the key | Cross-user bleed (the security failure) | The runtime-API rule: outside + args |
| A checkout with cached prices "because the catalog is cached" | SHAPE 4 (fully request-scoped) ignored | Prices at checkout are *computed* server-side per submit (module 12's checkout); the catalog cache is a different surface with a different contract |
| "Static is faster, let's make it all static" | The incomplete sentence | The four shapes; freshness is a dial, not a flag |

## 6. Security Notes

- The static shell is **public by definition** (it's on the CDN): anything in it is visible to everyone, forever (until invalidation). User-specific data in a shell is a leak — the model *prevents* it (user data is keyed per-user or request-scoped), but a *design* mistake (caching a "my org" read without orgId in the key) produces a public shell of private data. The module-20 ownership rule, enforced at the shell boundary.
- Build-time rendering runs your code **at build**: a `use cache`'d function that reads the DB at build needs DB access in your build environment (CI!) — and a *buggy* build-time read can poison the shell until deploy/invalidation. Fail builds loud on build-time read errors (don't let a broken query produce a broken shell).
- Dynamic holes are where request-time *validation* happens (session, tenancy) — the shell is served *before* those checks for public pages; that's why public shells must contain nothing gated (module 10-05: the authed shell is *per-session*, not global).

## 7. Performance Notes

- **TTFB = shell delivery.** With the shell on the CDN (+ prefetched), TTFB is an edge-cache hit regardless of how many holes stream behind it. This is the measurable claim: LCP on a shell-cached route ≈ CDN latency; the holes improve *progressive* metrics, not LCP (if the LCP element is *inside* a hole, that's a design bug — put the LCP element in the shell; module 18-05).
- **Build time is a deployment cost**: every `generateStaticParams` value and every shell render is paid at deploy. The head/tail discipline (module 07) is a *build-time* performance decision, not just a correctness one.
- **Cold holes are your real latency**: the common case is warm (cached); the cold case (first request post-deploy, a new slug) is the one your Suspense skeletons and error boundaries must make *graceful* — that's module 25's dashboard, and module 26's case study prices it.

## 8. Exercise

**Beginner.** Take the §4 product page into your scaffold (stub services). Force the three states: (a) a head slug (in `generateStaticParams`) — verify build-time render (build output lists it; the dev server shows it as static); (b) a tail slug — verify the first request is slower (filling the entry) and the second is a cache hit (logs); (c) the stock chip — verify it streams *after* the hero (waterfall). Screenshot all three.

**Intermediate.** Write the decision table (the §3 four questions) for **every read** in your dashboard page (the module-19 page: session, revenue, orders, activity). For each: the shape (1–4), the profile, the shell-eligibility, the invalidation. Rows where you can't answer shell-eligibility are rows that need a design decision — make it, in writing.

**Production.** The **LCP audit**: for each capstone page, name the LCP element and answer: is it in the shell, or in a hole? For every LCP-in-a-hole, either move it (make the data shell-eligible — is the life short *for that element*, or for the whole section?) or accept the latency with a number (measure it: hole latency for the LCP element, cold and warm). This is the performance module's (18-05) warm-up, and the answer is always "the LCP element gets the shell's treatment."

## 9. Architecture Challenge

**Prompt:** A blog post page needs: the article body (published once, rarely edited), the author bio (changes when the author updates their profile — weekly), a "related posts" list (changes with every publish — daily), a live "reading now" counter (changes continuously), and the post's OG image (generated once at publish).

Produce the full caching document: per read — shape, profile, tag/invalidation, shell-eligibility, and the *deploy* behavior (what happens to a 2-year-old post's page when you deploy a new version of the site). Then: what does the author see when they update their bio (immediate? when? which primitive?), and what does the "reading now" counter's `seconds` life *cost* on a viral post (10k readers/min) — do you keep it cached, make it a client-side estimate, or kill it? Defend.

<details>
<summary>Model answer</summary>
Per read:
- Body: SHAPE 1 — `use cache`, `cacheLife('max')`, `cacheTag('post:'+slug)` — shell, CDN, deploy-invalidated (new build ID → new keys → re-filled on first hit; a 2-year-old post's body re-renders once per deploy — *that's the deploy behavior*: build ID in the key means deploys are a global invalidation; for a rarely-edited body that's fine, the re-render is one query per post per deploy, lazily).
- Author bio: SHAPE 2 — `cacheLife('weeks')` (weekly changes), `cacheTag('author:'+authorId)` — shell-eligible (weeks ≥ storable) — invalidated by the author's profile-save action (`updateTag('author:'+id)` for the author, SWR for the fleet).
- Related posts: `cacheLife('days')`, `cacheTag('posts')` — shell-eligible; every publish revalidates `'posts'` → related lists across *all* posts refresh in the background over the next hour of traffic (SWR spread — no storm, module 23).
- OG image: generated at publish (a build/publish-time artifact stored as an asset — not a per-request compute; `metadata.openGraph.images` points at the static URL, module 14).
- Reading now: `seconds`-cached is a *dynamic hole* — on a viral post (10k readers/min), each reader's request triggers… nothing (the entry is seconds-stale → background regen ~1/min is deduped — the *cost* is one count query/min, trivial). The *real* cost is elsewhere: the counter's *value* is a lie at any granularity (it's a sample, not a count) — and a 30s-stale "1,247 reading now" is indistinguishable from live for the user. Keep it `seconds` (the hole is cheap; the dedupe does the work). If the counter's *source* were expensive (a live socket count), switch to a client-side estimate (each viewer increments a shared Redis counter on page-view; the read is O(1)) — but the caching shape (hole, seconds) stays the same.
What the author sees on a bio update: their *own* profile page updates immediately (the save action's `updateTag('author:'+id)` — read-your-own-writes); *blog posts they appear on* refresh in the background (SWR) — the author sees the old bio on *other people's posts* for up to the stale window; if that's unacceptable (their bio is brand-sensitive), the save action also `revalidateTag('posts-by-author:'+id, 'max')` — but the *author's own* view is the `updateTag` guarantee. The distinction (actor vs fleet) is the module-23 rule applied.
</details>

## 10. Official Documentation

- Caching (Cache Components): https://nextjs.org/docs/app/getting-started/caching
- `cacheLife` (prerendering behavior, short-lived → holes): https://nextjs.org/docs/app/api-reference/functions/cacheLife
- ISR with Cache Components (App Shells): https://nextjs.org/docs/app/guides/incremental-static-regeneration-cache-components
- Building (what `next build` does): https://nextjs.org/docs/app/guides/building
- CDN Caching: https://nextjs.org/docs/app/guides/cdn-caching

## 11. What You Should Know Before Continuing

- [ ] I can state the four canonical shapes and classify any page as a combination
- [ ] I can explain build-time vs request-time *precisely* (what each can/can't read)
- [ ] I can finish the sentence "static is faster ___" (for the freshness you can afford — per read, not per page)
- [ ] I know the deploy behavior (build ID in the key = global invalidation, lazily re-filled)
- [ ] I've done the LCP audit: every page's LCP element is shell-eligible, or the latency is measured and accepted

**Next:** Module 25 — the Dashboard Caching Case Study (the capstone's per-section caching plan, with the reasoning written out).
