# Module 23 — Revalidation: Tags, `updateTag`, `revalidateTag`, `revalidatePath`

**Phase 5: Caching · Module 23 of 101**

> **Where does this run?** Tag *assignment* happens `[SERVER]` inside `use cache` scopes (`cacheTag`). Tag *invalidation* happens `[SERVER]` in the code that performs the mutation — Server Functions, Route Handlers, or background jobs. This is the WHO question of module 20, with mechanics.

---

## 1. Concept — Invalidation is a *design artifact*, not a call

Time-based revalidation (`cacheLife`) handles "the data changes on its own." **On-demand revalidation** handles "I changed it, now make it fresh." The primitives (official model):

| Primitive | Where you may call it | Behavior | The contract it expresses |
|---|---|---|---|
| `cacheTag(tag)` | inside a `use cache` scope | *marks* the entry with a tag (tags are how entries are found) | "I am invalidatable by X" |
| `updateTag(tag)` | **Server Functions (actions) only** | **immediately expires** the tag's entries | *read-your-own-writes*: the actor sees the change at once |
| `revalidateTag(tag, profile)` | Server Functions **and** Route Handlers | **stale-while-revalidate**: mark stale; next request serves old + regenerates in background; `profile` = how long stale may be served during the background window | "others may see it a moment later; freshness is a background job" |
| `revalidatePath(path)` | Server Functions and Route Handlers | invalidates *all cached data for that route path* | the escape hatch when you don't know the tags — **prefer tags** (official guidance: more precise, avoids over-invalidation) |

**The design rule (the module's thesis):** every mutation in the capstone has a *written* invalidation spec — which tag(s), which primitive (immediate vs SWR), and *why*. The spec lives next to the mutation (the action file's header comment) and in the caching inventory. "The cache is probably fine" is not a spec.

**Why two tag primitives?** Because the *actor* and *everyone else* have different freshness needs:

```
You cancel an order.
  YOU (the actor):      must see 'cancelled' NOW        → updateTag (immediate)
  EVERYONE ELSE (who has the cached view):               → revalidateTag (SWR)
       sees 'pending' until their next visit regenerates in the background —
       acceptable: a status change is not a correctness event for *them*
       (and their "order list" cache life is minutes-hours anyway)
```

The Server Function can call `updateTag('orders')` *and* `revalidateTag('orders', 'max')` — the official pattern is to do the immediate for the actor and let SWR handle the fleet. (In practice `updateTag` in the action + the framework's SWR on the invalidated entries covers both; the distinction matters most when the *mutation happens outside a Server Function* — a webhook Route Handler can only `revalidateTag`, which is exactly right: the *user* who paid doesn't need their own view instantly, the *system* does.)

## 2. Mental Model — the invalidation topology (who touches what)

```mermaid
flowchart LR
    subgraph MUTATIONS["Mutations (the WHO)"]
        A1["cancelOrder (Server Function)"]
        A2["publishProduct (Server Function)"]
        WH["payment webhook (Route Handler)"]
        JOB["nightly analytics job (external)"]
    end
    subgraph TAGS["Tags (the WHAT)"]
        T1["'orders'"]
        T2["'order:{id}'"]
        T3["'products' / 'product:{slug}'"]
        T4["'analytics'"]
    end
    A1 -->|updateTag + revalidateTag| T1
    A1 -->|revalidateTag| T2
    A2 -->|updateTag (actor) + revalidateTag (fleet)| T3
    WH -->|revalidateTag| T1
    WH -->|revalidateTag| T2
    JOB -->|revalidateTag (via admin route or direct call)| T4
```

**Tag design rules:**
1. **Tag by resource type + stable identifier** (`'product:' + slug`), never by PII or by anything that changes (`'product:' + id` is fine; `'product:' + userEnteredName` is a key-churn disaster).
2. **Coarse + fine, deliberately:** `'products'` (the lists) + `'product:{slug}'` (the detail). A publish invalidates both — the list *and* the affected detail. Invalidation is *additive* (you may invalidate more than the minimum); staleness is not.
3. **One owner per tag** (the module-20 WHO question): exactly one place in the codebase knows that `'orders'` must be invalidated — the order mutation module. Two places = two divergent lists = one forgotten.
4. **The webhook is a mutation** (it changes state) — so it participates in invalidation, from its Route Handler (hence `revalidateTag`'s route-handler permission; `updateTag` stays action-only because "immediate for the actor" is an *actor* concept).

## 3. Production Code

### 3.1 Tagged reads (the capstone, collected)

`FILE: src/services/orders.ts` (excerpt — [SERVER])

```ts
export async function getOrderListCached(orgId: string, input: OrderListInput) {
  'use cache'
  cacheLife('minutes')                       // lists: frequently glanced
  cacheTag('orders')                          // coarse: any order mutation
  return listOrders(orgId, input)             // the uncached core (module 17)
}

export async function getOrderCached(id: string, orgId: string) {
  'use cache'
  cacheLife('hours')
  cacheTag('orders')
  cacheTag('order:' + id)                     // fine: this specific order
  return getOrderById(id, orgId)
}
```

### 3.2 The mutation with its invalidation spec

`FILE: src/features/order/order-actions.ts` (production pattern — [SERVER])

```ts
'use server'

import { headers } from 'next/headers'
import { redirect } from 'next/navigation'
import { updateTag, revalidateTag } from 'next/cache'
import { auth } from '@/lib/auth'
import { z } from 'zod'
import { cancelOrderService } from '@/services/orders'
import { AppError, isAppError } from '@/lib/errors'

/**
 * INVALIDATION SPEC (module 23):
 *   mutation:      order → cancelled (status transition, audited in service)
 *   tags touched:  'orders' (lists) · 'order:{id}' (this detail)
 *   actor:         updateTag → immediate (read-your-own-writes; redirect to the fresh page)
 *   fleet:         revalidateTag('orders','max') + revalidateTag('order:{id}','max')
 *                  → SWR: other viewers refresh in background on next hit
 */
export async function cancelOrder(id: string): Promise<void> {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })

  const parsed = z.string().uuid().parse(id)
  await cancelOrderService(session.organizationId, parsed)   // service enforces tenancy + allowed transition

  updateTag('orders')
  updateTag('order:' + parsed)
  revalidateTag('orders', 'max')
  revalidateTag('order:' + parsed, 'max')
  // redirect(`/orders/${parsed}?cancelled=1`)  // optional: land on the fresh detail
}
```

### 3.3 The webhook (Route Handler — `revalidateTag` only, and that's correct)

`FILE: src/app/api/webhooks/stripe/route.ts` (production pattern — [SERVER])

```ts
import { NextResponse } from 'next/server'
import { revalidateTag } from 'next/cache'
import { verifyStripeSignature } from '@/services/stripe'     // signature check FIRST (module 19-02)
import { markOrderPaid } from '@/services/orders'
import { z } from 'zod'

const eventSchema = z.object({
  id: z.string(),
  type: z.string(),
  data: z.object({ object: z.object({ id: z.string(), metadata: z.record(z.string()).optional() }) }),
})

export async function POST(req: Request) {
  const signature = req.headers.get('stripe-signature')
  const body = await req.text()
  const event = verifyStripeSignature(body, signature)   // throws on bad signature → 400
  if (!event) return NextResponse.json({ error: 'invalid signature' }, { status: 400 })

  const parsed = eventSchema.safeParse(event)
  if (!parsed.success) return NextResponse.json({ error: 'malformed event' }, { status: 400 })

  if (parsed.data.type === 'payment_intent.succeeded') {
    const orderId = parsed.data.data.object.metadata?.orderId
    if (orderId) {
      await markOrderPaid(orderId)                        // service: tenancy-aware status transition
      // NO 'updateTag' here (action-only) — SWR is the right contract for "the payer's
      // next view is fresh within a beat; the fleet is fresh in the background":
      revalidateTag('orders', 'max')
      revalidateTag('order:' + orderId, 'max')
    }
  }
  return NextResponse.json({ received: true })            // 2xx fast: Stripe retries on failure
}
```

**The 2xx-fast rule:** webhooks should ACK before heavy work if possible (Stripe retries non-2xx aggressively). If `markOrderPaid` is fast (it's a status update), do it inline; if the event fans out (emails, reports), do the status update inline and *queue* the rest (module 22-04 background jobs).

### 3.4 `revalidatePath` — the escape hatch, used honestly

`FILE: src/features/organization/org-actions.ts` (simplified example — [SERVER])

```ts
'use server'
// Renaming the organization's public display name touches a dozen cached fragments
// that were tagged inconsistently during a fast build-out. Until the tags are
// audited (todo: tag audit, module 25), invalidate the *paths* that show it:
import { revalidatePath } from 'next/cache'

export async function updateOrgName(formData: FormData) {
  // …validate + mutate…
  revalidatePath('/dashboard')
  revalidatePath('/admin')
  // Documented escape hatch: broad, visible, and temporary. The audit ticket is the exit.
}
```

## 4. Common Mistakes (the invalidation catalog)

| Mistake | Symptom | Fix |
|---|---|---|
| Mutation without any invalidation | The module-19 ticket: "admin sees paid, customer sees pending" | The invalidation spec (written, next to the mutation); the inventory row |
| `updateTag` from a Route Handler | Build/runtime error (actions-only) — or, in older models, silently wrong | `revalidateTag` from handlers; reserve `updateTag` for actor paths |
| Tagging the read but invalidating a *different* tag ("typo tag") | Eternal staleness — the tag never matches | Tags as a **typed constant module** (`src/lib/tags.ts`: `export const TAGS = { orders: 'orders', order: (id: string) => 'order:' + id } as const`) — no string literals in two places |
| Invalidating *everything* after every mutation (`revalidatePath('/')`) | Cold-cache storms on the whole app after any click | Per-mutation specs; the escape hatch is *documented and temporary* (§3.4) |
| Two owners for one tag | One owner's refactor forgets the invalidation the other relied on | One owner per tag (rule 3); grep for the tag should find one writer + the readers |
| Relying on SWR for the *actor* | "I saved, but I still see the old value" — the actor's own view is the one that *must* be immediate | `updateTag` in the action (actor) + SWR for the fleet |
| Invalidating *before* the write commits | A concurrent request regenerates from the *old* DB row and re-caches it | Write (committed) → invalidate, in that order; the service's transaction boundary is the checkpoint (module 09-04) |

**The typed tags module** (production pattern — [BOTH-safe values, used [SERVER]):

`FILE: src/lib/tags.ts`

```ts
// The single source of truth for cache tag names.
// Readers: services (cacheTag). Writers: actions/webhooks/jobs (revalidate*).
// Adding a tag here without a writer = a review finding (module 20: who invalidates it?).
export const TAGS = {
  products: 'products',
  product: (slug: string) => `product:${slug}`,
  orders: 'orders',
  order: (id: string) => `order:${id}`,
  pricing: 'pricing',
  catalog: 'catalog',
  revenue: (orgId: string) => `revenue:${orgId}`,
  analytics: 'analytics',
  featuredTeams: 'featured-teams',
} as const
```

## 5. Security Notes

- **Stale authorization is a bypass** (module 20, repeated as the security line here): a mutation that changes *permissions* (role change, suspension, org removal) must invalidate everything those permissions gated — and the *session-derived* reads are request-scoped (uncached) so the next request is correct regardless. The danger is the *UI affordance* cached in a `use cache`'d component: "this user can cancel" rendered into a cached fragment outlives the suspension. Rule: **capability-rendered UI is either request-scoped or invalidated by the permission mutation** — the inventory must show it.
- Webhooks are *inbound mutations from the internet*: signature verification before *any* state change or invalidation (a forged webhook that revalidates tags is a cheap DoS — revalidation storms are measurable load).
- `revalidatePath` breadth = information breadth: a mass path invalidation after a *failed* admin action (that rolled back) tells the fleet "something changed" when nothing did — invalidate only on committed mutations.

## 6. Performance Notes

- **SWR is the load-shaper:** `revalidateTag(tag, 'max')` spreads regeneration across *requests* (one per expired entry's next hit, deduped) — a hammer invalidation (`revalidatePath('/')`) turns the next traffic minute into a fleet-wide cold start. The `profile` argument is how you bound the stale window during that spread (the official semantics: how long stale may be served while fresh generates).
- Invalidating *fine-grained* tags (`'order:{id}'`) keeps the blast radius one entry; coarse tags (`'orders'`) cover the lists. Both, on a cancel — cheap and precise.
- The *ordering* (write → invalidate) plus the transaction boundary is what makes concurrent-regeneration safe (a regeneration that started before the commit re-caches old data, *then* the invalidation marks it stale — the sequence self-heals; the reverse order can cache the old value *after* the invalidation passed).

## 7. Exercise

**Beginner.** Create `src/lib/tags.ts` with the 9 tags above. Migrate every `cacheTag('literal')` in your services to `TAGS.*`. Grep: zero string-literal tags outside `tags.ts`. (This is the whole exercise — and it's the difference between a cache you can debug and one you can only guess at.)

**Intermediate.** Implement `cancelOrder` (§3.2) + the webhook (§3.3) against the stub order service. Script the scenario: (a) user cancels → their *next* view is immediate (updateTag), (b) a second "user" (different cookie, same org, cached list from before) → their next visit shows fresh within the SWR window (log the regeneration), (c) the webhook marks another order paid → both tags invalidated. Log a line per step: tag, primitive, resulting entry state.

**Production.** The **tag audit**: for every tag in `tags.ts`, answer in the inventory: readers (which functions `cacheTag` it), writers (which mutations invalidate it), owner (one module name), and the failure mode if a writer is deleted. Find at least one tag with no writer (if you have none, add a "documentation-only" tag you *deliberately* leave writer-less and watch the audit catch it). This audit is module 25's debugging lab's first case, pre-empted.

## 8. Architecture Challenge

**Prompt:** The "suspend user" admin action must take effect *immediately*: the suspended user's open dashboard tab should stop working, their API access should stop, and their cached "is admin" fragments must go. Design the full invalidation: which reads are request-scoped (and why that's most of the answer), which *are* cached and must be tagged, what the action invalidates in what order, what the suspended user sees on their next action (401? 403? 404? — pick and defend), and what you do about their *session* itself (hint: module 10-03 — the session row, not the cache).

<details>
<summary>Model answer</summary>
Most of the answer is *architecture, not invalidation*: session resolution is request-scoped (module 10-05) and the session row carries the suspension flag — so the *next* request from the suspended user fails at the layout's `getSession` check, **before** any cached fragment is consulted. Their open tab's *next action* (a Server Function POST) hits the action's session check → rejected. Cached UI fragments don't matter if the *entry* to the authenticated area is closed: the layout 401s/redirects.
What *is* cached and must be tagged: (1) any **public** surface that renders user data (a public profile page showing "admin" — tag `'profile:'+id`, invalidated on suspend), (2) admin lists that include the user (`'admin-users'` — so the admin console stops showing them as active), (3) *their org's* dashboard fragments keyed by org (they're out of the org — `'revenue:{orgId}'` etc. — but those are org-scoped by membership; suspending the user changes *their* access, not the org's data — the org's cache legitimately stays warm for the other members).
Action order: ① transaction: set `users.suspendedAt`, revoke the user's *membership rows* (org access), **delete (or expire) the user's session rows** (module 10-03: immediate revocation — the session store is the mechanism, not the cache); ② `updateTag('profile:'+id)` + `revalidateTag('admin-users','max')` (actor/admin immediate; fleet SWR).
What they see on the next action: **401** (identity-level: "you are not who you were" — the session is gone, module 11-03's 401-vs-403: suspension is an *identity/lifecycle* event, not a permission denial; 403 would be for "known user, may not do *this*"). Their open tab: the in-flight/next request 401s → the client router (or the action's thrown 401) redirects to /login with a "your account was suspended" message (the login page reads the flag server-side to show it — don't leak the reason via the redirect URL alone).
The session revocation is the *primary* control; the tag invalidations cover the *public* surfaces the session check can't reach (their profile is publicly readable). Both, in one action, written spec.
</details>

## 9. Official Documentation

- Revalidating: https://nextjs.org/docs/app/getting-started/revalidating
- `revalidateTag`: https://nextjs.org/docs/app/api-reference/functions/revalidate-tag
- `updateTag`: https://nextjs.org/docs/app/api-reference/functions/update-tag
- `revalidatePath`: https://nextjs.org/docs/app/api-reference/functions/revalidate-path
- `cacheTag`: https://nextjs.org/docs/app/api-reference/functions/cache-tag
- How Revalidation Works: https://nextjs.org/docs/app/guides/how-revalidation-works

## 10. What You Should Know Before Continuing

- [ ] I can state the four primitives' permissions (where callable) and behaviors (immediate vs SWR vs path)
- [ ] I can write an invalidation spec (tags, primitives, actor vs fleet, order) for any mutation
- [ ] My tags live in one typed module with one owner each
- [ ] I know the write→invalidate ordering and why it's a concurrency property
- [ ] I can design a permission-change invalidation (the suspend challenge) with the session row as the primary control

**Next:** Module 24 — Static vs Dynamic Rendering: the "why, not 'static is faster'" module (build time vs request time, and how caching changes the equation).
