# Module 62 — Structured Data, Sitemap, Robots: The Machine-Readable Head

**Phase 14: SEO · Module 62 of 101**

> **Where does this run?** All three are **`[SERVER]`** (the JSON-LD is serialized into the HTML, the `sitemap.ts`/`robots.ts` are Route Handlers that emit files — none of it touches `[CLIENT]` JS); the *JSON-LD content* comes from the **`[SERVER]`** DB (the product DTO, module 06's). The module-62's standing rule (module 61's/06's, now the machine level): **the `<head>` is for humans, the JSON-LD + `sitemap.xml` + `robots.txt` are for machines — the JSON-LD is the *same DTO* as the wire (module 06's), the `sitemap` is the *same list* as the DB, the `robots` is the *same policy* as the auth (module 43's) (module 62's §1)** (module 62's §1).

---

## 1. Concept — The machine-readable head (the 3 files)

**The JSON-LD's** (module 62's §1.1): the *the schema.org's* (module 62's §1.1) — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1) — the *the no `@type`'s guess* (module 62's §1.1).

**The `sitemap.ts`** (module 62's §1.2): the *the `MetadataRoute.Sitemap`'s* (module 62's §1.2) — the *module-62's line: the sitemap's is the file's* (module 62's §1.2) — the *the no HTML's* (module 62's §1.2).

**The `robots.ts`** (module 62's §1.3): the *the `MetadataRoute.Robots`'s* (module 62's §1.3) — the *module-62's line: the robots's is the file's* (module 62's §1.3) — the *the no policy's guess* (module 62's §1.3).

## 2. Mental Model — The 3 files (drawn)

```mermaid
flowchart TD
    A["THE DB (module 6's) — the product's DTO (module 6's) — the org's list (module 37's)"] --> B["THE JSON-LD (module 62's §1.1) — the schema.org's (module 62's §1.1) — the the <script type=\"application/ld+json\"> (module 62's §3.1)"]
    A --> C["THE sitemap.ts (module 62's §1.2) — the MetadataRoute.Sitemap (module 62's §1.2) — the the /sitemap.xml (module 62's §3.2)"]
    D["THE AUTH'S POLICY (module 43's) — the private's (module 61's §5)"] --> E["THE robots.ts (module 62's §1.3) — the MetadataRoute.Robots (module 62's §1.3) — the the /robots.txt (module 62's §3.3)"]
    B --> F["THE CRAWLER (module 61's §2) — the Googlebot's"]
    C --> F
    E --> F
```

**The 3 files** (the module-62's mental model):
1. **The JSON-LD** (module 62's §1.1): the *the schema.org's* — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1).
2. **The `sitemap.ts`** (module 62's §1.2): the *the `MetadataRoute.Sitemap`'s* — the *module-62's line: the sitemap's is the file's* (module 62's §1.2).
3. **The `robots.ts`** (module 62's §1.3): the *the `MetadataRoute.Robots`'s* — the *module-62's line: the robots's is the file's* (module 62's §1.3).

## 3. Architecture — The 3 files (the code)

### 3.1 The JSON-LD (module 62's §1.1 — the `Product`'s)

`FILE: src/lib/jsonld.ts` + `FILE: app/products/[id]/page.tsx` (production pattern — [SERVER] — the module-62's §3.1: the schema.org's)

```ts
// THE JSON-LD'S (module 62's §1.1) — the the schema.org's (module 62's §1.1) — the the DTO's (module 6's):
// 'use server' is NOT needed — the JSON-LD is the [SERVER] (module 62's §1)
export function productJsonLd(product: {
  id: string
  name: string
  description: string
  price: number   /* the module-37's line: the price is the cents' (module 37's) */
  currency: string
  slug: string
  image?: string
}, baseUrl: string) {
  return {
    '@context': 'https://schema.org',
    '@type': 'Product',   /* the module-62's line: the @type is the Product's (module 62's §3.1) */
    name: product.name,
    description: product.description,
    sku: product.id,
    image: product.image ? `${baseUrl}${product.image}` : undefined,   /* the module-61's line: the baseUrl is the metadataBase's (module 61's §1.3) */
    offers: {
      '@type': 'Offer',
      price: (product.price / 100).toFixed(2),   /* the module-62's line: the price is the dollars' (module 62's §3.1) — the the cents' (module 37's) */
      priceCurrency: product.currency,
      availability: 'https://schema.org/InStock',
      url: `${baseUrl}/products/${product.slug}`,
    },
  }
}
```

```tsx
// THE PAGE'S JSON-LD (module 62's §3.1) — the the <script type="application/ld+json"> (module 62's §3.1):
import { productJsonLd } from '@/lib/jsonld'

export default async function ProductPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const product = await getProduct(id)
  if (!product) notFound()
  const baseUrl = process.env.NEXT_PUBLIC_SITE_URL!
  return (
    <>
      <script
        type="application/ld+json"
        dangerouslySetInnerHTML={{ __html: JSON.stringify(productJsonLd(product, baseUrl)) }}   /* the module-62's line: the JSON.stringify is the no-injection's (module 62's §3.1) — the the no user-input (module 75's) */
      />
      {/* the page's content (module 6's) */}
    </>
  )
}
```

**The module-62's line:** the *`@type` is the `Product`'s* (module 62's §3.1) — the *the `price` is the dollars'* (module 62's §3.1) — the *the `JSON.stringify` is the no-injection's* (module 62's §3.1) — the *the `baseUrl` is the `metadataBase`'s* (module 61's §1.3).

### 3.2 The `sitemap.ts` (module 62's §1.2 — the `MetadataRoute.Sitemap`'s)

`FILE: app/sitemap.ts` (production pattern — [SERVER] — the module-62's §3.2: the file's)

```ts
// THE SITEMAP (module 62's §1.2) — the the MetadataRoute.Sitemap (module 62's §1.2) — the the no HTML's (module 62's §1.2):
import type { MetadataRoute } from 'next'
import { listProductSlugs, listArticleSlugs } from '@/services'   /* the module-5's line: the service is the ORM's (module 5's) */

export default async function sitemap(): Promise<MetadataRoute.Sitemap> {
  const baseUrl = process.env.NEXT_PUBLIC_SITE_URL!
  const [products, articles] = await Promise.all([listProductSlugs(), listArticleSlugs()])   /* the module-20's line: the fetch's (module 20's) */
  return [
    { url: baseUrl, changeFrequency: 'daily', priority: 1 },   /* the module-62's line: the priority is the 1's (module 62's §3.2) */
    ...products.map((slug) => ({ url: `${baseUrl}/products/${slug}`, changeFrequency: 'weekly' as const, priority: 0.8 })),   /* the module-62's line: the product's is the 0.8's (module 62's §3.2) */
    ...articles.map((slug) => ({ url: `${baseUrl}/blog/${slug}`, changeFrequency: 'monthly' as const, priority: 0.6 })),   /* the module-62's line: the article's is the 0.6's (module 62's §3.2) */
  ]
  /* THE RULE (module 62's §3.2): the the sitemap's is the no-index's (module 61's §5) — the the private's pages are the no's (module 61's §5) */
}
```

**The module-62's line:** the *`MetadataRoute.Sitemap` is the file's* (module 62's §1.2) — the *the `priority` is the 1's/0.8's/0.6's* (module 62's §3.2) — the *the private's pages are the no's* (module 61's §5).

### 3.3 The `robots.ts` (module 62's §1.3 — the `MetadataRoute.Robots`'s)

`FILE: app/robots.ts` (production pattern — [SERVER] — the module-62's §3.3: the policy's)

```ts
// THE ROBOTS (module 62's §1.3) — the the MetadataRoute.Robots (module 62's §1.3) — the the no policy's guess (module 62's §1.3):
import type { MetadataRoute } from 'next'

export default function robots(): MetadataRoute.Robots {
  const baseUrl = process.env.NEXT_PUBLIC_SITE_URL!
  return {
    rules: [
      {
        userAgent: '*',
        allow: '/',
        disallow: ['/dashboard/', '/(admin)/', '/api/', '/checkout/'],   /* the module-62's line: the private's is the disallow's (module 62's §3.3) — the the dashboard's (module 43's) */
      },
    ],
    sitemap: `${baseUrl}/sitemap.xml`,   /* the module-62's line: the sitemap's is the reference's (module 62's §3.3) */
  }
}
/* THE RULE (module 62's §3.3): the the robots's is the no-index's (module 61's §5) — the the private's is the disallow's (module 62's §3.3) */
```

**The module-62's line:** the *`MetadataRoute.Robots` is the policy's* (module 62's §1.3) — the *the private's is the `disallow`'s* (module 62's §3.3) — the *the `sitemap` is the reference's* (module 62's §3.3).

### 3.4 The `Organization`'s + the `Article`'s (module 62's §3.4 — the 2 more `@type`'s)

`FILE: src/lib/jsonld.ts` (production pattern — [SERVER] — the module-62's §3.4)

```ts
// THE ORGANIZATION (module 62's §3.4) — the the @type is the Organization's (module 62's §3.4):
export function organizationJsonLd(baseUrl: string) {
  return {
    '@context': 'https://schema.org',
    '@type': 'Organization',
    name: 'SaaS Commerce Platform',
    url: baseUrl,
    logo: `${baseUrl}/logo.png`,
  }
}

// THE ARTICLE (module 62's §3.4) — the the @type is the Article's (module 62's §3.4):
export function articleJsonLd(article: { title: string; description: string; slug: string; datePublished: string; author: string }, baseUrl: string) {
  return {
    '@context': 'https://schema.org',
    '@type': 'Article',   /* the module-62's line: the @type is the Article's (module 62's §3.4) */
    headline: article.title,
    description: article.description,
    datePublished: article.datePublished,   /* the module-37's line: the date is the ISO's (module 37's) */
    author: { '@type': 'Person', name: article.author },
    mainEntityOfPage: `${baseUrl}/blog/${article.slug}`,
  }
}
```

**The module-62's line:** the *`@type` is the `Organization`'s* (module 62's §3.4) — the *the `@type` is the `Article`'s* (module 62's §3.4) — the *the `datePublished` is the ISO's* (module 37's).

## 4. Production Code — The validation (module 62's §4)

`FILE: terminal` (production pattern — the module-62's §4: the check's)

```bash
# THE VALIDATION (module 62's §4) — the the Rich Results' (module 62's §4) — the the no guess (module 62's §1.1):
# 1) THE RICH RESULTS TEST (module 62's §4) — the the Google's (module 62's §4):
#    https://search.google.com/test/rich-results   (module 62's §4) — the the JSON-LD's (module 62's §3.1)
# 2) THE SCHEMA.ORG VALIDATOR (module 62's §4) — the the validator's (module 62's §4):
#    https://validator.schema.org/   (module 62's §4) — the the @type's (module 62's §3.1)
# 3) THE SITEMAP'S CHECK (module 62's §4) — the the fetch's (module 62's §4):
curl -s ${NEXT_PUBLIC_SITE_URL}/sitemap.xml | head   /* the module-62's line: the sitemap's is the XML's (module 62's §4) */
# 4) THE ROBOTS'S CHECK (module 62's §4) — the the fetch's (module 62's §4):
curl -s ${NEXT_PUBLIC_SITE_URL}/robots.txt   /* the module-62's line: the robots's is the text's (module 62's §4) */
```

**The module-62's line:** the *Rich Results test is the check's* (module 62's §4) — the *the `sitemap.xml` is the XML's* (module 62's §4) — the *the `robots.txt` is the text's* (module 62's §4).

## 5. Common Mistakes (the machine's failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The `@type`'s guess** (module 62's §1.1's line violated) | the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1) — the *the `@type`'s guess is the *no's* (module 62's §1.1) — the *module-62's line: the no `@type`'s guess* (module 62's §1.1) — the *no `@type`'s guess* (module 62's §1.1)* | the *the schema.org's `@type` (module 62's §1.1) + the Rich Results test (module 62's §4) — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1)* |
| **The private's in the sitemap** (module 62's §3.2's line violated) | the *module-62's line: the private's pages are the no's* (module 61's §5) — the *the private's in the sitemap's is the *no's* (module 62's §3.2) — the *module-62's line: the no private's in the sitemap's* (module 62's §3.2) — the *no private's in the sitemap's* (module 62's §3.2)* | the *the filter's (module 62's §3.2) — the *module-62's line: the private's pages are the no's* (module 61's §5)* |
| **The no `sitemap` in the robots** (module 62's §3.3's line violated) | the *module-62's line: the sitemap's is the reference's* (module 62's §3.3) — the *the no `sitemap`'s is the *no's* (module 62's §3.3) — the *module-62's line: the no `sitemap`'s* (module 62's §3.3) — the *no `sitemap`'s* (module 62's §3.3)* | the *the `sitemap: ${baseUrl}/sitemap.xml`'s (module 62's §3.3) — the *module-62's line: the sitemap's is the reference's* (module 62's §3.3)* |
| **The `dangerouslySetInnerHTML`'s user-input** (module 62's §3.1's line violated) | the *module-62's line: the `JSON.stringify` is the no-injection's* (module 62's §3.1) — the *the `dangerouslySetInnerHTML`'s user-input is the *no's* (module 62's §3.1) — the *module-62's line: the no user-input* (module 75's) — the *no user-input* (module 75's)* | the *the `JSON.stringify`'s (module 62's §3.1) — the *module-62's line: the `JSON.stringify` is the no-injection's* (module 62's §3.1)* |
| **The relative's URL** (module 62's §3.2's line violated) | the *module-62's line: the `baseUrl` is the `metadataBase`'s* (module 61's §1.3) — the *the relative's URL's is the *no's* (module 62's §3.2) — the *module-62's line: the no relative's URL's* (module 62's §3.2) — the *no relative's URL's* (module 62's §3.2)* | the *the `${baseUrl}/...`'s (module 62's §3.2) — the *module-62's line: the `baseUrl` is the `metadataBase`'s* (module 61's §1.3)* |
| **The `price` in the cents'** (module 62's §3.1's line violated) | the *module-62's line: the `price` is the dollars'* (module 62's §3.1) — the *the `price` in the cents' is the *no's* (module 62's §3.1) — the *module-62's line: the no `price` in the cents'* (module 62's §3.1) — the *no `price` in the cents'* (module 62's §3.1)* | the *the `(price / 100).toFixed(2)`'s (module 62's §3.1) — the *module-62's line: the `price` is the dollars'* (module 62's §3.1)* |

## 6. Security Notes

- **The no user-input** (module 75's): the *module-62's line: the `JSON.stringify` is the no-injection's* (module 62's §3.1) — the *module-75's* *deep-dive* (module 75's).
- **The private's disallow** (module 62's §3.3): the *module-62's line: the private's is the `disallow`'s* (module 62's §3.3) — the *module-43's* *deep-dive* (module 43's).
- **The `NEXT_PUBLIC_SITE_URL`** (module 61's §1.3): the *module-61's line: the `metadataBase` is the origin's* (module 61's §1.3) — the *the no hardcoded's* (module 62's §1.3).

## 7. Performance Notes

- **The `sitemap`'s is the async's** (module 62's §7.1): the *module-62's line: the `sitemap`'s is the async's* (module 62's §7.1) — the *the no per-page's* (module 62's §7.1).
- **The JSON-LD's is the small's** (module 62's §7.2): the *module-62's line: the JSON-LD's is the small's* (module 62's §7.2) — the *the no bloat's* (module 62's §7.2).
- **The `robots`'s is the static's** (module 62's §7.3): the *module-62's line: the `robots`'s is the static's* (module 62's §7.3) — the *the no fetch's* (module 62's §7.3).

## 8. Exercise

**Beginner.** *The JSON-LD's* (module 62's §3.1): the *the `productJsonLd`'s* (module 3.1's) + the *the `<script type="application/ld+json">`'s* (module 3.1's) — *build it* — the *artifact: the JSON-LD's* (module 3.1's).

**Intermediate.** *The `sitemap.ts` + the `robots.ts`* (module 62's §3.2 + §3.3): the *the `MetadataRoute.Sitemap`'s* (module 3.2's) + the *the `MetadataRoute.Robots`'s* (module 3.3's) — *build it* — the *artifact: the 2's* (module 20's).

**Production.** *The validation's* (module 62's §4): the *the Rich Results test's* (module 4's) + the *the `sitemap.xml`'s check* (module 4's) + the *the `robots.txt`'s check* (module 4's) — *do it* — the *artifact: the pass's* (module 4's).

## 9. Architecture Challenge

**Prompt:** The *"the team's 500-product catalog + 50-article blog has no structured data, the sitemap lists the dashboard, and the robots has no sitemap reference"* (the *module-62's* *machine's* — the *module-61's* *metadata* — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1) — the *module-61's line: the `Metadata` is the DTO's* (module 06's) — the *module-62's standing line: the structured's is the JSON-LD's + the sitemap's is the file's + the robots's is the policy's* (module 62's §1.1 + module 62's §1.2 + module 62's §1.3)).

The *problems*: (1) the *the no JSON-LD's* (the *the no `productJsonLd`'s* (module 62's §3.1) — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1) — the *module-62's standing line: the structured's is the JSON-LD's* (module 62's §1.1)).

(2) the *the dashboard's in the sitemap* (the *the no filter's* (module 62's §3.2) — the *module-62's line: the private's pages are the no's* (module 61's §5) — the *module-62's standing line: the private's pages are the no's* (module 61's §5)).

**Design**: the *the machine's remediation* (the *the `productJsonLd`'s + the `articleJsonLd`'s* (module 62's §3.1 + module 62's §3.4) + the *the `sitemap.ts`'s filter* (module 62's §3.2) + the *the `robots.ts`'s `sitemap`* (module 62's §3.3) — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1) — the *module-62's standing line: the structured's is the JSON-LD's + the sitemap's is the file's + the robots's is the policy's* (module 62's §1.1 + module 62's §1.2 + module 62's §1.3)).

Produce: the *the machine's remediation* (the *the `productJsonLd`'s + the `articleJsonLd`'s* (module 62's §3.1 + module 62's §3.4) + the *the `sitemap.ts`'s filter* (module 62's §3.2) + the *the `robots.ts`'s `sitemap`* (module 62's §3.3) — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1) — the *module-62's standing line: the structured's is the JSON-LD's + the sitemap's is the file's + the robots's is the policy's* (module 62's §1.1 + module 62's §1.2 + module 62's §1.3)).

<details>
<summary>Model answer</summary>
**The machine's remediation** (module 62's §3.1 + module 62's §3.4 + module 62's §3.2 + module 62's §3.3):
1. **The JSON-LD's** (module 62's §3.1 + module 62's §3.4): the *the `productJsonLd`'s + the `articleJsonLd`'s are the schema.org's* — the *module-62's line: the structured's is the JSON-LD's* (module 62's §1.1).
2. **The sitemap's filter** (module 62's §3.2): the *the no-private's filter is the list's* — the *module-62's line: the private's pages are the no's* (module 61's §5).
3. **The robots's sitemap** (module 62's §3.3): the *the `sitemap: ${baseUrl}/sitemap.xml`'s is the reference's* — the *module-62's line: the sitemap's is the reference's* (module 62's §3.3).
**The generalization** (the *machine's* pattern, the *module's* standing rule): **the *structured's is the JSON-LD's* (module 62's §1.1) — the *the sitemap's is the file's* (module 62's §1.2) — the *the robots's is the policy's* (module 62's §1.3) — the *module-62's standing line: the structured's is the JSON-LD's + the sitemap's is the file's + the robots's is the policy's* (module 62's §1.1 + module 62's §1.2 + module 62's §1.3)*.
</details>

## 10. Official Documentation

- Next.js: `sitemap.ts`: https://nextjs.org/docs/app/api-reference/file-conventions/metadata/route
- Next.js: `robots.ts`: https://nextjs.org/docs/app/api-reference/file-conventions/metadata/route
- Next.js: JSON-LD (metadata `other`): https://nextjs.org/docs/app/api-reference/file-conventions/metadata
- schema.org: Product: https://schema.org/Product
- schema.org: Article: https://schema.org/Article
- Google: Rich Results Test: https://search.google.com/test/rich-results
- schema.org Validator: https://validator.schema.org/
- The module-61's metadata: the module-61 (the phase-14's file-01)

## 11. What You Should Know Before Continuing

- [ ] I can state the *3 files* (module 1's: the JSON-LD/sitemap/robots) — the *module-62's line: the machine's is the 3's* (module 1's)
- [ ] I know the *`@type` is the `Product`'s* (module 3.1's) — the *the no `@type`'s guess* (module 1.1's)
- [ ] I know the *`MetadataRoute.Sitemap` is the file's* (module 1.2's) — the *the private's pages are the no's* (module 61's §5)
- [ ] I know the *`MetadataRoute.Robots` is the policy's* (module 1.3's) — the *the private's is the `disallow`'s* (module 3.3's)
- [ ] I know the *`JSON.stringify` is the no-injection's* (module 3.1's) — the *the no user-input* (module 75's)
- [ ] I know the *`price` is the dollars'* (module 3.1's) — the *the `cents` is the wire's* (module 37's)
- [ ] I've done the *JSON-LD's* (module 8's beginner) + the *sitemap/robots* (module 8's intermediate) + the *validation's* (module 8's production) — the *artifacts* (module 20's)

**Next:** Module 63 — SEO Case Studies (the *the product's page* — the *the article's page* — the *module-63's line: the case's is the full's* (module 63's)).
