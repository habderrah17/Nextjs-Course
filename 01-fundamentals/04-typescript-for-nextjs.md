# Module 04 — TypeScript for Professional Next.js

**Phase 1: Foundations · Module 4 of 101 (closes Phase 1)**

> **Where does this run?** TypeScript is a *compile-time* tool: it runs on your machine and in CI, never in production. Its job in Next.js is to make the server/client boundary, the data flow, and the API contracts **loud failures at build time** instead of runtime incidents.

---

## 1. Concept — Compile-time safety vs runtime validation (the most important TS distinction in this course)

TypeScript checks **what you wrote**. It cannot check **what arrives at runtime** — network bodies, URLs, env files, database rows, form `FormData`. The professional posture:

```
TS types  →  describe the contract (what SHOULD exist)
Zod       →  enforce the contract at every trust boundary (what ACTUALLY exists)
```

Trust boundaries in a Next.js app: (1) browser form/URL input, (2) env vars, (3) external API responses, (4) data returned by your own ORM when it crosses into a DTO, (5) Server Function arguments (the client is adversarial — module 07-01). You will *derive* TS types from Zod (`z.infer`) so the contract is defined exactly once.

## 2. Mental Model — the type flow

```
Zod schema (source of truth for shape)
   │  z.infer
   ▼
DTO types (types/ or schemas/)  ──→  Service signatures (services/)
   │                                    │
   ▼                                    ▼
Props at the boundary                Server Function args
(server → client components)         (validated inside the action, again with Zod)
   │
   ▼
UI rendering (React's props typing)
```

Rules: (a) shapes that cross a trust boundary are Zod-first, types derived; (b) internal-only shapes can be plain TS types; (c) the boundary's *props* are typed by inference, never re-declared by hand.

## 3. The feature set, with production code

### 3.1 Type-safe props & polymorphic components

`FILE: src/components/page-header.tsx` (production pattern — [BOTH])

```tsx
import type { ReactNode } from 'react'

type PageHeaderProps = {
  title: string
  description?: string
  actions?: ReactNode
}

export function PageHeader({ title, description, actions }: PageHeaderProps) {
  return (
    <header className="flex flex-wrap items-center justify-between gap-4">
      <div>
        <h1 className="text-2xl font-semibold tracking-tight">{title}</h1>
        {description && <p className="text-sm text-muted-foreground">{description}</p>}
      </div>
      {actions && <div className="flex items-center gap-2">{actions}</div>}
    </header>
  )
}
```

Polymorphic `asChild`-style pattern (as shadcn components use it):

```tsx
import { Slot } from '@radix-ui/react-slot'
import type { ComponentPropsWithoutRef, ElementType } from 'react'

type ButtonVariant = 'default' | 'outline' | 'ghost' | 'destructive'
type ButtonSize = 'sm' | 'md' | 'lg'

export type ButtonProps<C extends ElementType = 'button'> =
  & ComponentPropsWithoutRef<C>
  & { asChild?: boolean; variant?: ButtonVariant; size?: ButtonSize }

export function Button<C extends ElementType = 'button'>({
  asChild = false,
  className,
  variant = 'default',
  size = 'md',
  ...props
}: ButtonProps<C>) {
  const Comp = asChild ? Slot : (props as { as?: C }).as ?? 'button'
  // variant/size → class map (module 13)
  return <Comp className={cn(baseClasses, variantClasses[variant], sizeClasses[size], className)} {...props} />
}

// Usage: <Button asChild><Link href="/pricing">Plans</Link></Button>
```

### 3.2 Generics where they earn their keep (the service layer)

`FILE: src/services/typed-list.ts` (production pattern — [SERVER])

```ts
/**
 * A typed, paginated list result. ONE definition used by every list service,
 * so UIs (and tests) share the same pagination contract.
 */
export type Page<T> = {
  items: T[]
  nextCursor: string | null
  total: number
}

/**
 * Cursor pagination input. `TFilter` is the filter shape for the domain.
 */
export type ListInput<TFilter> = {
  cursor?: string
  limit?: number
  filter: TFilter
  sort?: { by: keyof TFilter extends never ? string : string; dir: 'asc' | 'desc' } | { by: string; dir: 'asc' | 'desc' }
}
```

`FILE: src/services/products.ts` (production pattern — [SERVER])

```ts
import { db } from '@/db'
import { products, orgs } from '@/db/schema'
import { and, desc, eq, ilike, lt, sql } from 'drizzle-orm'
import type { ListInput, Page } from './typed-list'
import type { ProductDTO } from '@/types/product'

// The filter shape is a Zod-inferred type (defined in features/product/product-schemas.ts):
import type { ProductFilter } from '@/features/product/product-schemas'

export async function listProducts(orgId: string, input: ListInput<ProductFilter>): Promise<Page<ProductDTO>> {
  const limit = Math.min(input.limit ?? 20, 100)
  const rows = await db
    .select()
    .from(products)
    .where(
      and(
        eq(products.orgId, orgId),          // tenancy — from session, never client (module 11)
        input.filter.name && ilike(products.name, `%${input.filter.name}%`),
        input.filter.status && eq(products.status, input.filter.status),
        input.cursor ? lt(products.id, input.cursor) : undefined,
      ),
    )
    .orderBy(desc(products.id))
    .limit(limit + 1)                        // fetch one extra → cursor exists?

  const hasMore = rows.length > limit
  const items = rows.slice(0, limit).map(productToDto)
  return { items, nextCursor: hasMore ? items.at(-1)!.id : null, total: Number(await db.select({ n: sql`count(*)` }).from(products).where(eq(products.orgId, orgId))) }
}

export function productToDto(p: typeof products.$inferSelect): ProductDTO {
  // $inferSelect = the ORM row type; the DTO strips/reshapes it at the service boundary.
  return { id: p.id, name: p.name, slug: p.slug, priceCents: p.priceCents, status: p.status, updatedAt: p.updatedAt.toISOString() }
}
```

Notes: `typeof products.$inferSelect` — Drizzle derives row types from the schema; the service returns **DTOs** (plain objects, `Date` → ISO string) so the boundary stays serializable and the UI never depends on ORM shapes.

### 3.3 Discriminated unions for errors and API responses

`FILE: src/types/api.ts` (production pattern — [BOTH])

```ts
/**
 * Every server-to-client failure is ONE of these. The `status` tag lets the
 * client switch exhaustively — a missing case is a compile error.
 */
export type ApiError =
  | { status: 400; code: 'validation'; fieldErrors: Record<string, string[]>; message: string }
  | { status: 401; code: 'unauthenticated'; message: string }
  | { status: 403; code: 'forbidden'; message: string }
  | { status: 404; code: 'not_found'; message: string }
  | { status: 409; code: 'conflict'; message: string }
  | { status: 422; code: 'business_rule'; message: string }
  | { status: 500; code: 'internal'; message: string }

export type ApiResult<T> =
  | { ok: true; data: T }
  | { ok: false; error: ApiError }

// Client usage — exhaustive:
function handle(result: ApiResult<unknown>) {
  if (result.ok) return
  switch (result.error.code) {
    case 'validation': showFieldErrors(result.error.fieldErrors); break
    case 'unauthenticated': router.replace('/login'); break
    // add a new case to ApiError without handling it → tsc fails. That is the point.
  }
}
```

### 3.4 Environment variable typing (fail fast, typed)

`FILE: src/schemas/env.ts` (production pattern — [SERVER])

```ts
import { z } from 'zod'

const server = z.object({
  DATABASE_URL: z.string().url().startsWith('postgresql://'),
  BETTER_AUTH_SECRET: z.string().min(32),
  BETTER_AUTH_BASE_URL: z.string().url().default('http://localhost:3000'),
  // Storage (module 16)
  STORAGE_PROVIDER: z.enum(['local', 's3']).default('local'),
  S3_BUCKET: z.string().optional(),
})

const publicEnv = z.object({
  NEXT_PUBLIC_APP_URL: z.string().url().default('http://localhost:3000'),
})

// Parsed ONCE at server boot. A missing var is a crash at boot (CI catches it),
// not a mystery 500 three requests later.
export const serverEnv = server.parse(process.env)
// Exported for client-safe access ONLY via the server-side `env` accessor in lib/env.ts;
// NEXT_PUBLIC_ values are already inlined by Next at build time — read them directly where needed.
```

### 3.5 Typed route params & searchParams (all async in 16.x)

`FILE: src/app/(marketing)/products/[slug]/page.tsx` (production pattern — [SERVER])

```tsx
import { notFound } from 'next/navigation'
import { z } from 'zod'
import { getProductBySlug } from '@/services/products'
import { ProductView } from '@/features/product/components/product-view'

const paramsSchema = z.object({ slug: z.string().min(1).max(200) })

export default async function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  const { slug } = await params                     // ← awaited: required in 16.x
  if (!paramsSchema.safeParse({ slug }).success) notFound()

  const product = await getProductBySlug(slug)      // null → notFound inside service? no: here
  if (!product) notFound()

  return <ProductView product={product} />
}
```

`FILE: src/app/(marketing)/products/page.tsx` (searchParams — [SERVER])

```tsx
import { z } from 'zod'
import { listProducts, type ProductFilter } from '@/services/products'

const searchSchema = z.object({
  q: z.string().max(100).optional(),
  status: z.enum(['draft', 'active', 'archived']).optional(),
  page: z.coerce.number().int().min(1).optional(),
  cursor: z.string().max(128).optional(),
  sort: z.enum(['newest', 'price_asc', 'price_desc']).default('newest'),
})

export default async function CatalogPage({ searchParams }: { searchParams: Promise<Record<string, string | string[] | undefined>> }) {
  const raw = await searchParams
  const flat: Record<string, string | undefined> = Object.fromEntries(
    Object.entries(raw).map(([k, v]) => [k, Array.isArray(v) ? v[0] : v]),
  )
  const parsed = searchSchema.safeParse(flat)
  const query = parsed.success
    ? parsed.data
    : { q: undefined, status: undefined, page: undefined, cursor: undefined, sort: 'newest' as const } // malformed URL → defaults, never a crash

  const filter: ProductFilter = { name: query.q, status: query.status }
  const result = await listProducts('default-org', { filter, cursor: query.cursor, limit: 24, sort: { by: query.sort, dir: 'desc' } })
  return <div>{/* ProductGrid + pagination links (module 17) */}</div>
}
```

### 3.6 Typed forms (Zod inference, shared both sides)

`FILE: src/features/user/profile-schemas.ts` (production pattern — [BOTH])

```ts
import { z } from 'zod'

export const profileFormSchema = z.object({
  name: z.string().trim().min(1, 'Name is required').max(120),
  email: z.email('Enter a valid email'),
  phone: z.string().trim().min(7).max(32).optional().or(z.literal('')),
})

export type ProfileFormValues = z.infer<typeof profileFormSchema>   // form state type
export type ProfileFormInput = z.input<typeof profileFormSchema>    // raw input type (coerced fields differ)
```

Client (RHF) and server (action) both import the same schema — the module 12 round-trip is type-safe end to end.

### 3.7 Utility types you will actually use

```ts
// types/product.ts  ([BOTH])
import type { ProductFormValues } from '@/features/product/product-schemas'

export type ProductDTO = {
  id: string
  name: string
  slug: string
  priceCents: number
  status: 'draft' | 'active' | 'archived'
  updatedAt: string
}

// From an existing shape:
type ProductPreview = Pick<ProductDTO, 'id' | 'name' | 'priceCents'>          // list rows
type ProductCreateInput = Omit<ProductFormValues, 'slug'>                     // slug is generated server-side
type ProductPatch = Partial<Omit<ProductCreateInput, 'id'>>                   // update forms
type ProductStatusMap = Record<ProductDTO['status'], string>                  // exhaustive label map
```

## 4. Common Mistakes

| Mistake | Fix |
|---|---|
| `any` as "I'll type it later" | `unknown` + a guard; `any` is a compile-time XSS — it disables the only safety you have |
| Trusting `formData.get('id')` as a string | It's `FormDataEntryValue \| null`; coerce + validate (Zod) before use |
| Re-declaring prop types by hand when Zod already inferred them | `z.infer` once, import everywhere |
| Typing `process.env` access ad hoc | One parser (`schemas/env.ts`), parsed once |
| Treating `params`/`searchParams` as sync objects (pre-15 muscle memory) | `await` them — in 16.x they are Promises, required |
| DTOs leaking ORM types into `types/` | DTOs are standalone value types; mapping lives in the service |
| `Record<string, any>` for env/headers | Typed accessors; headers are typed as `Headers` |

## 5. Security Notes

- The client is untrusted: **every Server Function re-validates its input** with the same Zod schema the form used (module 07-03). TS types in the action signature are documentation, not enforcement.
- `unknown` at trust boundaries is a security practice: you cannot accidentally trust a parsed value without going through the schema.

## 6. Performance Notes

- `zod/v4` compiles fast (100× fewer tsc instantiations than Zod 3 for big schemas) — don't fear per-request validation; a schema parse is microseconds. (If you ever measure otherwise, you're parsing the wrong thing.)
- Keep DTO types small: prop type size affects type-check time, not runtime.

## 7. Exercise

**Beginner.** In your scaffold, create `src/types/api.ts` with the `ApiError`/`ApiResult` union. Write a client function that handles `ApiResult<never>` exhaustively; add a new variant and confirm tsc forces you to handle it.

**Intermediate.** Define the Product domain end-to-end with types: `product-schemas.ts` (Zod: `productFormSchema`, `productFilterSchema`), `types/product.ts` (DTO + derived types), `services/products.ts` (stub `listProducts` returning `Page<ProductDTO>` from an in-memory array). Every signature must be inferred, none hand-typed where inference works.

**Production.** Add `schemas/env.ts` to the scaffold with the 6 vars from §3.4. Delete `DATABASE_URL` from `.env.local` and confirm: (a) local boot fails with a readable zod error naming the var, (b) CI would fail identically. Write the error message down — it is your future onboarding doc.

## 8. Architecture Challenge

**Prompt:** Your team's "typed" API returns `{ success: boolean; data?: any; message?: string }` from every endpoint. The client does `if (res.success) use(res.data)`.

1. List the concrete failure modes this contract has (think: validation errors, 401s, races, new error kinds).
2. What is the migration path that doesn't break existing callers overnight?
3. Where in *this* course's stack would the new contract's type live, and who derives it?

<details>
<summary>Model answer</summary>
1. `data: any` erases every type safety downstream; `message?` conflates user-facing copy with debugging; success/failure can't express 401 vs 403 vs 409, so the client can't react (redirect vs alert vs retry); a new failure kind ships as `success: false, message: 'weird thing'` and every client silently mis-handles it; field-level validation errors have no shape, so forms can't auto-map them.
2. Ship the discriminated union *additively* (new field `error?: ApiError`), have the server populate both the old fields and the new one for one release, migrate clients to read the new shape, then deprecate the old fields. Behind a versioned prefix (`/api/v2/...`) if you can't coordinate.
3. `src/types/api.ts` ([BOTH] — pure data, cross-boundary safe), with `ApiResult<T>` derived from it; servers produce it, RHF/action round-trips consume it (module 12).
</details>

## 9. Official Documentation

- TypeScript with Next.js: https://nextjs.org/docs/app/getting-started/installation (TypeScript section)
- TypeScript handbook: https://www.typescriptlang.org/docs/handbook/2/index.html
- Zod 4: https://zod.dev
- React type reference (props): https://react.dev/reference/react
- Drizzle type inference: https://orm.drizzle.team/docs/usage/types

## 10. What You Should Know Before Continuing

- [ ] I can state compile-time vs runtime validation and name the 5 trust boundaries
- [ ] I can write a generic `Page<T>`/`ListInput<TFilter>` and use them in a service
- [ ] My error types are a discriminated union with exhaustive handling
- [ ] Env vars are Zod-parsed at server boot; I've seen the failure mode
- [ ] I `await` `params` and `searchParams` without thinking
- [ ] Form schemas are Zod-first with `z.infer` types used on both sides

**Phase 1 complete.** Capstone Stage 1 is green: scaffold, structure, TS discipline, CI. **Next:** Phase 2 — Routing. Module 05: The App Router, deeply.
