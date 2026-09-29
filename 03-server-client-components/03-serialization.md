# Module 14 — Serialization: The Exact Rules of the Boundary

**Phase 3: Server/Client Components · Module 14 of 101**

> **Where does this run?** Serialization happens `[BOTH — BOUNDARY]`: the server serializes props going **down** to client components (RSC payload), and arguments going **up** to Server Functions are serialized with RSC argument rules. It is the most *enforced-by-compiler* part of the boundary — violations are build errors, which is a gift.

---

## 1. Concept — Two serialization systems, one rule of thumb

The React docs define **two** serialization regimes, and Next.js uses both at the boundary:

| Direction | System | Can cross |
|---|---|---|
| **Server → client** (props) | RSC **props** serialization | Primitives, `Date`, `Map`/`Set` (limited), `BigInt`, functions **only as Server Function references**, client component references, `Promise` (via `use`), errors, class *instances* only if they're plain-data (rarely) |
| **Client → server** (action args) | `use server` **argument** serialization (stricter) | Primitives, `Date`, `Map`/`Set`, `BigInt`, `FormData`, plain objects/arrays — **no** functions, **no** class instances, **no** `undefined` in some positions, **no** `Symbol`/`RegExp` |

**Rule of thumb for 95% of decisions: pass *plain JSON-compatible data* (DTOs), pass *Server Function references* for callbacks, and if you need to pass anything else, that's a design smell.**

The official reference (linked below) is the contract; this module gives you the working version with the traps.

## 2. Mental Model — the DTO is the boundary protocol

```
SERVER                                    CLIENT
─────────────────────────────────────────────────────────────
ORM row (Drizzle $inferSelect)
  → productToDto(row)      ← the mapping IS the serialization discipline
  → ProductDTO { id: string, name: string, priceCents: number,
                 updatedAt: string (ISO), … }
  → JSON-safe, timezone-explicit, no methods, no undefined traps
  → crosses as props
```

Why DTOs *are* the serialization protocol (not just a style choice):

1. **They make the wire contract explicit** — what the client receives is defined by `types/*.ts`, reviewed in PRs, and stable while the schema evolves.
2. **They kill the traps** — `Date` → ISO string (client formats); `null` vs `undefined` (DTOs use `null`); bigints (→ strings/numbers deliberately); relational objects (flattened to what the UI renders).
3. **They decouple** — a schema migration that changes a column doesn't ripple into client components; the service's mapper is the blast radius.

## 3. The traps (each one has cost you in production somewhere)

| Trap | What happens | The fix |
|---|---|---|
| `undefined` in a prop | RSC serialization rejects `undefined` in many positions (the wire format can't distinguish "no value" from "missing") | DTOs use `null`; `?? null` at the mapper |
| `Date` object | Serializes, but the client re-parses it in *its* timezone → SSR/client formatting mismatch → hydration mismatch | ISO strings in DTOs; one shared formatter `[BOTH]` |
| Functions (non-action) | Build error: "Only plain objects, arrays, and iterables may be passed as props" | Pass a Server Function reference, or inline the data the function would have computed |
| Class instances (non-React) | Build error or silent `null` | Plain objects at the mapper |
| `RegExp`, `Symbol`, `Buffer`, `File` (server-side) | Not serializable | Reconstruct on the client from primitives (`new RegExp(str)`) |
| Passing a server component as a child to a client prop | Client can't import server components | Restructure: the server renders `<ClientIsland>{serverContent}</ClientIsland>` — children-as-props work, *imports* don't |
| Cyclic references | Serialization fails | DTOs are trees, not graphs; break cycles in the mapper |
| Huge prop payloads (whole tables) | Bloats the RSC payload + re-sends on every soft navigation | Paginate; pass IDs and let the client fetch detail only when expanded (deliberate client fetch, module 13 challenge) |

## 4. Production Code

### 4.1 The mapper (the discipline in code)

`FILE: src/services/orders.ts` (excerpt — [SERVER])

```ts
import { db } from '@/db'
import { orders, customers } from '@/db/schema'
import { and, desc, eq, ilike } from 'drizzle-orm'
import type { OrderDTO, OrderStatus } from '@/types/order'

const STATUS_SET: Record<string, OrderStatus> = {
  pending: 'pending',
  paid: 'paid',
  shipped: 'shipped',
  cancelled: 'cancelled',
  refunded: 'refunded',
} as const

export function orderToDto(o: typeof orders.$inferSelect): OrderDTO {
  // Deliberate, explicit mapping — the compiler forces completeness (no spread-and-pray).
  return {
    id: o.id,
    number: o.orderNumber,
    status: STATUS_SET[o.status] ?? 'pending',     // unknown DB value → safe default, never undefined
    customerName: o.customerName ?? 'Unknown customer',
    totalFormatted: formatCents(o.totalCents),     // format ONCE, server-side (module 13)
    createdAt: o.createdAt.toISOString(),          // Date → ISO
    canCancel: o.status === 'pending',            // server decides capability (UI just renders)
  }
}

export async function listOrders(orgId: string, opts: { limit: number; cursor?: string }): Promise<{ items: OrderDTO[]; nextCursor: string | null }> {
  const rows = await db
    .select()
    .from(orders)
    .where(and(eq(orders.orgId, orgId), opts.cursor ? lt(orders.id, opts.cursor) : undefined))
    .orderBy(desc(orders.id))
    .limit(opts.limit + 1)
  const hasMore = rows.length > opts.limit
  const items = rows.slice(0, opts.limit).map(orderToDto)
  return { items, nextCursor: hasMore ? items.at(-1)!.id : null }
}
```

`FILE: src/types/order.ts` ([BOTH])

```ts
export type OrderStatus = 'pending' | 'paid' | 'shipped' | 'cancelled' | 'refunded'

export type OrderDTO = {
  id: string
  number: string
  status: OrderStatus
  customerName: string
  totalFormatted: string
  createdAt: string            // ISO — the client renders, never re-derives
  canCancel: boolean           // capability computed server-side
}
```

### 4.2 Passing an action across the boundary (the only "function" that crosses)

`FILE: src/features/order/order-actions.ts` ([SERVER], file-level `'use server'`)

```ts
'use server'

import { headers } from 'next/headers'
import { auth } from '@/lib/auth'
import { updateTag } from 'next/cache'
import { cancelOrderService } from '@/services/orders'
import { z } from 'zod'

export async function cancelOrder(id: string): Promise<void> {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) throw new Error('Unauthorized')

  // Argument validation — the client is adversarial (module 07-03 pattern).
  const parsed = z.string().uuid().parse(id)

  await cancelOrderService(session.organizationId!, parsed)   // service enforces tenancy + status
  updateTag('orders')                                         // read-your-own-writes (module 05-04)
}
```

The client (§4 of module 13) calls `cancelOrder(order.id)` — `order.id` (a string) serializes fine; the *promise* it returns resolves client-side; and the RSC re-render after `updateTag` updates the row without any manual state sync.

### 4.3 `use()` for promises crossing to the client

A server component can pass a **Promise** as a prop; the client unwraps it with `use()` inside a Suspense boundary:

`FILE: src/app/(app)/orders/[id]/page.tsx` (simplified example — [SERVER])

```tsx
import { use, Suspense } from 'react'
import { getOrderById } from '@/services/orders'
import { OrderTimeline } from '@/features/order/components/order-timeline'

function Timeline({ orderId }: { orderId: string }) {
  const order = use(getOrderById(orderId))   // client: suspends until resolved
  return <OrderTimeline order={order} />
}

export default function OrderPage({ params }: { params: Promise<{ id: string }> }) {
  const { id } = use(params)
  return (
    <div>
      <OrderHeaderSkeletonless id={id} />
      <Suspense fallback={<TimelineSkeleton />}>
        <Timeline orderId={id} />
      </Suspense>
    </div>
  )
}
```

(`use(params)` is the React 19 idiom for awaited params in a render — module 07 showed the `await` version; both are correct, `use` composes with Suspense.)

## 5. Common Mistakes

| Mistake | Fix |
|---|---|
| `JSON.parse(JSON.stringify(dto))` "to make it serializable" | You've hidden the trap (Dates become strings anyway, `undefined` vanishes) *and* lost type safety — fix the DTO instead |
| Passing the ORM row "for now" | The moment a column is renamed, the client breaks silently; the mapper is the seam that should break |
| `canCancel` computed on the client from status | The client's status is a stale snapshot; capability decisions are server-side (DTO field) |
| Receiving `searchParams` and passing the raw object down | `searchParams` values are `string \| string[] \| undefined` — flatten + parse (module 10) before the boundary |
| Using `undefined` as "no value" in DTOs | `null`; it's a wire-format constraint, not a style opinion |

## 6. Security Notes

- The mapper is a **sanitization point**: the client should never receive more than it renders. A row with `internalCost` in the DTO is a leak even if the UI doesn't show it (it's in the RSC payload, inspectable).
- Action arguments are untrusted (module 07-03): Zod-parse at the top of every `'use server'` function. The serialized-ness of the argument is *not* validation.

## 7. Performance Notes

- DTO size = payload size = re-navigation cost (every soft navigation re-sends the props). Paginate; truncate; use IDs + on-demand detail.
- `formatCents` server-side means the client ships *zero* formatting code and renders faster (no `Intl.NumberFormat` setup on the hot path).
- Memoization note: with React Compiler on (module 02-02), stable prop identity matters less for re-renders — but the *wire* cost is unchanged; DTO size is a network fact, not a render fact.

## 8. Exercise

**Beginner.** Take one service from the capstone stubs. Write its `toDto` mapper *without* a spread operator — every field assigned explicitly. Add a column to the schema (e.g., `internalCostCents`) and confirm the compiler *forces* you to decide whether it crosses. Write that decision in a comment.

**Intermediate.** Build a component that receives a `Date` prop and renders it with `toLocaleDateString()` — on the server (a server component) and on the client (an island). Set your machine's TZ to UTC and run the server with TZ=America/New_York. Observe the hydration mismatch. Fix it the DTO way (ISO + one formatter) and confirm clean hydration.

**Production.** Create `scripts/dto-audit.mjs` (static analysis, grep-level): flags any `services/*.ts` export that returns `typeof ….$inferSelect` without a `toDto` mapper, and any client component whose props include a type imported from `db/schema`. Run it on the capstone; it should be green by the end of Phase 4.

## 9. Architecture Challenge

**Prompt:** The product detail page needs to render a **specification sheet** (50+ key/value rows) stored as JSONB in the DB. The client needs it for a "compare products" feature (kept in a client store while the user checks 3 products).

Options: (a) pass the full 50-row object in the DTO; (b) pass only the rendered subset; the compare feature fetches the full spec via a small client fetch to a Route Handler; (c) stream the spec in a Suspense hole and pass the full object to the client store on hydrate.

Choose, with the wire-cost numbers in mind (the spec JSON is ~15KB). Where does the compare feature's data *actually* come from in your answer?

<details>
<summary>Model answer</summary>
(b) — with a twist: the DTO carries the *rendered* spec (rows the page shows, ~4KB after formatting) as before; the compare feature is the one *legitimate client fetch* (module 13 challenge logic): a Route Handler `GET /api/v1/products/:id/spec` returns the full 15KB spec, fetched *only when the user opens compare mode* (not on page load). (a) costs 15KB × every navigation to that product (it re-sends on each soft navigation), for a feature 2% of users touch. (c) "on hydrate" is the worst: it couples the store to hydration timing, and the 15KB still rides the RSC payload. The Route Handler is the BFF case (module 08): a data *resource* a client feature needs on demand. Note the asymmetry that makes this defensible: the page's spec is *view data* (server-rendered, cached), the compare spec is *feature data* (client lifecycle, on-demand) — two different data classes, two different paths.
</details>

## 10. Official Documentation

- RSC: what can be serialized (props): https://react.dev/reference/rsc/use-client#serializable-types
- `use server`: serializable arguments: https://react.dev/reference/rsc/use-server#serializable-parameters-and-return-values
- `use`: https://react.dev/reference/react/use
- use cache: serialization requirements: https://nextjs.org/docs/app/api-reference/directives/use-cache#serialization
- Hydration errors: https://nextjs.org/docs/messages/hydration-error

## 11. What You Should Know Before Continuing

- [ ] I can state the two serialization regimes (props down, args up) and the plain-JSON rule of thumb
- [ ] My services return DTOs with explicit mappers; `null` not `undefined`; ISO dates; formatted strings
- [ ] I know the 8 traps and the fix for each
- [ ] I can explain why the mapper is a security + performance + schema-decoupling seam
- [ ] I can identify the legitimate client-fetch case (on-demand feature data via a Route Handler)

**Next:** Module 15 — Server-First Architecture: the BAD → GOOD transformations, and when SPA-style is actually right.
