# Module 07 — Dynamic & Catch-All Routes

**Phase 2: Routing · Module 7 of 101**

> **Where does this run?** Segment matching is `[SERVER]` at request time; `params` is a **Promise you must await** (required since Next 15, enforced in 16). What a dynamic route does with its value — prerender, render per request, or stream — is a caching decision (module 05).

---

## 1. Concept — Static, dynamic, and "both"

- **Static segment** (`/pricing`): the URL is known at build time → prerenderable.
- **Dynamic segment** (`/products/[slug]`): the URL value arrives with the request → the page must know how to handle *any* slug.
- The modern (16.x, Cache Components) model dissolves the old "static page vs dynamic page" dichotomy: a dynamic route can have a **static shell** (everything within a `cacheLife`) with **dynamic holes** (unknown slugs, runtime data), plus `generateStaticParams` to prerender *known* values at build time. This is PPR (Partial Prerendering) completing the story started in 2023.

## 2. Mental Model — the three questions for every dynamic route

1. **Which values exist?** Finite & known (product list, blog posts) → `generateStaticParams`. Infinite or user-driven (arbitrary user IDs, search slugs) → none.
2. **What should happen for an unknown value?** `notFound()` (404) is the default-correct answer. Never render an empty shell for a non-existent resource.
3. **What can the shell show before the dynamic value resolves?** With Cache Components: the layout chrome, skeletons, any `use cache`'d data. The unknown value's data streams in a Suspense hole or resolves as a dynamic render.

## 3. Architecture — matching & resolution

```
/products/electric-kettle
        │
        ▼
[slug] = "electric-kettle"
        │
        ├── generateStaticParams provided & slug is in the list?
        │       → prerendered HTML (build time) — served from CDN/shell
        │
        ├── slug not in the list (or none provided)?
        │       → request-time render:
        │           params = await … (Promise)
        │           product = await getProductBySlug(slug)   [SERVER]
        │           product == null → notFound() → 404 (nearest not-found.tsx)
        │           else → stream HTML (dynamic holes behind Suspense)
```

**Catch-alls** (`[...slug]`) and **optional catch-alls** (`[[...slug]]`):

| Pattern | Matches | `params` value |
|---|---|---|
| `/docs/[...slug]` | `/docs/a`, `/docs/a/b` (one or more) | `{ slug: ['a'] }`, `{ slug: ['a','b'] }` |
| `/docs/[[...slug]]` | `/docs` too | `{ slug: [] }` for the bare path |

Matching precedence: static > dynamic > catch-all, left to right. A `[[...slug]]` at the root (`app/[[...all]]`) is a *danger* — it swallows every URL (see anti-patterns).

## 4. Production Code

`FILE: src/app/(marketing)/products/[slug]/page.tsx` (production pattern — [SERVER])

```tsx
import type { Metadata } from 'next'
import { notFound } from 'next/navigation'
import { Suspense } from 'react'
import { z } from 'zod'
import { getProductBySlug } from '@/services/products'
import { ProductHero } from '@/features/product/components/product-hero'
import { ProductGallery } from '@/features/product/components/product-gallery'
import { ProductSkeleton } from '@/components/product-skeleton'

const paramsSchema = z.object({ slug: z.string().min(1).max(200).regex(/^[a-z0-9-]+$/) })

// Prerender known slugs at build time. The service reads from the DB at build —
// for a real catalog this is the published set; stale entries self-heal via revalidation (module 05).
export async function generateStaticParams() {
  const slugs = await listPublishedProductSlugs()
  return slugs.map((slug) => ({ slug }))
}

export async function generateMetadata({ params }: { params: Promise<{ slug: string }> }): Promise<Metadata> {
  const { slug } = await params
  const product = await getProductBySlug(slug)
  if (!product) return {}
  return {
    title: product.name,
    description: product.description,
    openGraph: {
      title: product.name,
      images: [{ url: product.imagePreviewUrl }],   // OG image — module 14
    },
  }
}

export default async function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  const parsed = paramsSchema.safeParse({ slug: (await params).slug })
  if (!parsed.success) notFound()                   // malformed slug → 404, not a crash
  const { slug } = parsed.data

  const product = await getProductBySlug(slug)
  if (!product) notFound()                           // unknown slug → 404

  return (
    <article>
      <ProductHero product={product} />
      {/* Slower data (related products, reviews) streams independently: */}
      <Suspense fallback={<ProductSkeleton />}>
        <RelatedProducts slug={slug} />
      </Suspense>
    </article>
  )
}
```

`FILE: src/app/(marketing)/blog/[...slug]/page.tsx` (simplified example — [SERVER])

```tsx
import { notFound } from 'next/navigation'
import { getPostBySlugs } from '@/services/posts'

export default async function BlogPost({ params }: { params: Promise<{ slug: string[] }> }) {
  const { slug } = await params
  const post = await getPostBySlugs(slug)            // handles ['2026', '09', 'title'] style hierarchies
  if (!post) notFound()
  return <article>{/* render post (module 14 SEO) */}</article>
}
```

**Typed validation of dynamic values** is a *security + robustness* habit, not over-engineering: slugs become query inputs. If `slug` flows into a Drizzle `eq()` it's safe (parameterized), but if it flows into a filename, URL, or SQL string — it's an injection surface. Validate at the edge of the segment (the schema above) and use the parsed value everywhere.

## 5. `generateStaticParams` with Cache Components (the current model)

- Provided values → prerendered at build into the static shell.
- Missing values → **App Shell** behavior: the shell is served immediately, the unknown param resolves at request time (ISR-with-Cache-Components pattern; see the official guide).
- Stale prerendered pages revalidate per their `cacheLife` and tags (module 05-04) — so "prerender at build" no longer means "frozen forever."

## 6. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `root/[[...all]]` "fallback page" | Swallows 404s — typos render your catch-all with junk data | Explicit `not-found` behavior; catch-alls only where a hierarchy genuinely exists (blog/docs) |
| Treating `params` as a sync object (pre-15 tutorials) | `params.slug` is `undefined`; crashes or silent wrong values | `await params` — always |
| Rendering a "product not found" UI instead of 404 | SEO poison: indexed 404-shaped pages, crawlers see 200 | `notFound()` |
| `generateStaticParams` returning *everything* (all slugs ever) | Build bloat; dead entries | Published set only; let revalidation + 404s handle the rest |
| Building the slug into a query string / file path | Injection (SSRF/path traversal) | Zod-validate the slug; treat it as untrusted input |
| Dynamic route for a value that's actually a query filter | `/products?category=x` becomes `/products/x` — URL semantics wrong | Filters are search params (module 05-06) |

## 7. Security Notes

- Dynamic values are **untrusted input**: schema-validate at the segment boundary; never interpolate into SQL/paths/URLs; whitelist slug shapes (regex above).
- IDOR (module 19-02): `/orders/[id]` must authorize the *owner* of the order in the service — a valid format ≠ a valid owner.

## 8. Performance Notes

- Known values prerendered = CDN-served = fastest tier. Design `generateStaticParams` for the *head* (popular 95%), let the tail stream.
- An uncached `getProductBySlug` on every request is a DB roundtrip per miss — tag it (`cacheTag('product:' + slug)`) with a `hours` life so repeat views are instant (module 05-04).

## 9. Exercise

**Beginner.** Add `/blog/[...slug]` with `generateStaticParams` returning three fake posts. Visit a listed slug (200) and an unlisted slug (404 via `notFound()`). In dev logs, confirm the listed ones were prerendered.

**Intermediate.** Give the product route a malicious slug tour: `/products/%2e%2e%2fsecret`, `/products/;drop`, `/products/abc/def`. With your slug regex, confirm all 404. Then remove the regex and *show* (in a comment) exactly where the value would become dangerous if it flowed into `fs` or a URL.

**Production.** Implement the App-Shell behavior for products: `generateStaticParams` for the top 10 published slugs; an uncached detail fetch behind Suspense for the rest. Measure (dev logs) the time-to-first-byte for a known vs unknown slug and record the delta.

## 10. Architecture Challenge

**Prompt:** You need `/u/[handle]` (public user profiles). Handles are user-chosen (arbitrary lowercase strings, must be unique, can be claimed/freed). A teammate proposes `generateStaticParams` for all 2M users.

1. What breaks at build time? At runtime for a freed handle?
2. Design the resolution: what's prerendered, what's streamed, what 404s, and how does a freed handle stop being indexed (work with the search team's constraints, stated as: "sitemaps are regenerated weekly")?
3. Where does handle *format* validation live, and why not in the service?

<details>
<summary>Model answer</summary>
1. Build time: generating 2M pages at build is a multi-hour build with a multi-GB output — the build becomes the bottleneck for every deploy, and 1.9M of them are never requested. Freed handle: the prerendered page still exists in the cache until it revalidates — a freed handle serves the *previous* owner's profile (privacy leak) until the tag revalidates.
2. Prerender a small head (top-N by traffic) with `cacheLife('weeks')` + `cacheTag('profile:'+handle)`; the tail streams on request; freed handle → service returns null → `notFound()` → 404 + the profile tag is revalidated in the same mutation that frees the handle (read-your-own-writes for the *admin* action; stale-while-revalidate for search indexers). Weekly sitemap excludes freed handles; `noindex` is not needed — a real 404 is.
3. Format validation (regex, length) at the route boundary (segment schema) — it's a *URL contract*; ownership/validity in the service. The boundary rejects what can't possibly be a handle before any I/O; the service answers the question only the database can answer.
</details>

## 11. Official Documentation

- Dynamic routes: https://nextjs.org/docs/app/building-your-application/routing/dynamic-routes
- `generateStaticParams`: https://nextjs.org/docs/app/api-reference/functions/generate-static-params
- `notFound`: https://nextjs.org/docs/app/api-reference/functions/not-found
- ISR with Cache Components: https://nextjs.org/docs/app/guides/incremental-static-regeneration-cache-components
- Caching (Cache Components): https://nextjs.org/docs/app/getting-started/caching

## 12. What You Should Know Before Continuing

- [ ] I can answer the three questions (which values / unknown behavior / shell contents) for any dynamic route
- [ ] I `await params` reflexively and validate slugs with Zod at the boundary
- [ ] I know static vs dynamic vs catch-all matching precedence
- [ ] I know the modern model: known values prerender, unknown values stream, stale values revalidate
- [ ] I can explain the freed-handle privacy failure and the tag-based fix

**Next:** Module 08 — `loading.tsx` / `error.tsx` / `not-found.tsx`: the special files that define every route's failure & loading behavior.
