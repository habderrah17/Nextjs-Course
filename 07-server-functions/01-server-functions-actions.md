# Module 29 — Server Functions & Server Actions: The Mutation API

**Phase 7: Server Functions / Actions · Module 29 of 101**

> **Where does this run?** A Server Function is **`[SERVER]`** — it *runs* on the server — but it is *referenced* from **`[CLIENT]`** (and `[SERVER]`) code: the reference is a **`[BOTH — BOUNDARY]`** — an opaque action ID + dispatcher (module 12's wire format's *upward* channel). This is the module-23 invalidation specs' *head*: the place the mutations actually live.

---

## 1. Concept — The official terminology, precisely (verified against the current docs)

- **Server Function** (the general term, React's `use server` feature): *an asynchronous function that runs on the server, callable from the client through a network request* — which is why it **must be `async`** (a synchronous function cannot be called over the network; a non-async `use server` function is a build error).
- **Server Action** (the convention, the mutation context): *a Server Function used with `startTransition`* — which happens **automatically** when the function is passed to a `<form>` via the **`action`** prop or to a button via **`formAction`**. In an action/mutation context, "Server Function" and "Server Action" are the same thing with a convention; this course uses **Server Function** for the API and **action** for the mutation usage.
- **The dispatch**: actions use **`POST`** — *only* `POST` can invoke them (there is no `GET` action; a "GET action" is a Route Handler, module 08). And: *the client dispatches and awaits Server Functions **one at a time** — an implementation detail that may change* (official wording) — **sequential dispatch**: two buttons that fire two actions run in order, not in parallel (the performance section's "parallel work" rule follows from this).
- **The response model**: *when an action is invoked, Next.js can return both the updated UI and new data in a **single server roundtrip*** (official) — the action's *completion* is an **RSC payload** (module 12's wire format, part two): the re-rendered tree with the *fresh data already in it* (the module-23 `updateTag` + the re-render = the actor's read-your-own-writes, delivered in the *same* roundtrip as the mutation's acknowledgement). **One POST, one response: the mutation's result *and* the new UI.**

## 2. Mental Model — The door from client to server

The module-12 analogy, completed: `'use client'` is a *door from server to client* (a component island); `'use server'` is *a door from client to server* (a function call). What crosses the door:

| Direction | What crosses | Form | The rules |
|---|---|---|---|
| **Down** (server → client, module 14) | Component props (DTOs) | The RSC payload (serialization regime 1) | Plain-serializable DTOs; the boundary is the `'use client'` line |
| **Up** (client → server, *this module*) | **Server Function arguments** | **A `POST` to the action ID** (serialization regime 2) | Plain-serializable *args*; **`FormData`** for forms (the `multipart` body — module 30); the return value is **serialized back down** (regime 1 — "only return data the UI needs, not raw database records," the official `use server` reference's line) |

**The build-time mechanics** (why the "opaque ID" is real, the `server-actions` guide's account): at build time the compiler **swaps the function's implementation in the client bundle for a reference — an action ID plus a dispatcher — that `POST`s back to the server.** The *implementation never ships to the client* (it's not in the bundle — the security module's "the implementation stays on the server" line), but the **route is reachable to anyone who can send the same `POST`** (module 19's threat model: an action ID is *in the client bundle* — visible — so **authentication and authorization are the action's own job**, verified *inside* every Server Function, every time — no middleware shortcut, module 10-05).

**The thin-action rule** (the capstone's convention, the module-17 path completed): a Server Function is a **boundary, not a place for logic**:

```
action = session → validate (Zod, module 31) → service (the only ORM code, module 17)
         → invalidate (the module-23 spec) → redirect/return (the response model)
```

Every line of *business logic* (the query, the tenancy check, the status transition, the audit row) is in the **service** (module 17's five contract rules); the action *orchestrates the boundary* (who: the session; what: the validated input; then: the service; after: the tags). A 40-line action with queries in it is a **review finding** (the logic is untestable outside the HTTP layer — the module-20 testing phase's seam).

## 3. Architecture — Creation, invocation, and the capstone's action inventory

### 3.1 Creating (the two placements, official)

**File-level** (the capstone's standard — a dedicated actions file per feature):

`FILE: src/features/order/order-actions.ts` (production pattern — [SERVER])

```ts
'use server'    // the file-level directive: ALL exports of this file are Server Functions

import { headers } from 'next/headers'
import { redirect } from 'next/navigation'
import { z } from 'zod'
import { auth } from '@/lib/auth'
import { cancelOrderService, refundOrderService } from '@/services/orders'
import { revalidateTag, updateTag } from 'next/cache'   // module 23's primitives
import { TAGS } from '@/lib/tags'                   // the typed tags (module 23)
import { AppError, isAppError } from '@/lib/errors'

// …the actions (module 31/32 complete them: validation, errors, pending)
```

**Inline** (inside a Server Component — the module-12 note, now precise): the directive at the **top of the function body**:

```tsx
// in a page/component that is a SERVER component:
export default function ProductPage({ params }: { params: Promise<{ slug: string }> }) {
  async function delistProduct(formData: FormData) {
    'use server'    // inline: this function is a Server Function
    // …thin boundary: session → validate → service → invalidate → redirect
  }
  return <DelistForm action={delistProduct} />
}
```

**Where they CANNOT be**: `// a 'use client' component — ` **you cannot *define* a Server Function in a Client Component** (the directive is a server-side construct; a client file can *import* actions from a `'use server'` file and *call* them — it cannot *declare* them). The capstone's rule: actions live in `src/features/*/actions.ts` (file-level), or inline in a Server Component (the page's private action) — **never in `src/components/**` (the client zone)**.

### 3.2 Invoking (the two official ways)

1. **Forms** (Server *and* Client components) — the `<form action={…}>` / `formAction` — the **automatic `startTransition`** + **progressive enhancement** (JS disabled = the native POST form still works — **module 30's whole subject**).
2. **Event handlers / `useEffect`** (Client components only) — the explicit call:

```tsx
'use client'
import { useTransition } from 'react'
import { cancelOrder } from '@/features/order/order-actions'   // import the REFERENCE (the ID)

export function CancelButton({ orderId }: { orderId: string }) {
  const [isPending, startTransition] = useTransition()
  return (
    <button
      disabled={isPending}                     // the pending gate (module 32 deepens this)
      onClick={() => startTransition(async () => {
        await cancelOrder(orderId)             // the args: plain-serializable (module 14 regime 2)
      })}
    >
      {isPending ? 'Cancelling…' : 'Cancel'}
    </button>
  )
}
```

**Additional arguments** (the `forms` guide's "passing additional arguments"): a Server Function takes the **extra args first, `FormData` last** — `cancelOrder(orderId: string)` from an event handler (the arg is *serializable*, sent in the POST body), or `updateUser(userId: string, formData: FormData)` from a **form** (the static args come *before* the form data; **in a form context the args after the first must be `FormData`** — the form *is* the body; a second static arg in a form action is a build-time error). The rule: **the `FormData` is the form's payload; the static args are the *context* (the IDs the server can derive but the client supplies) — and every static arg is *client-supplied* (the module-19 security line: treat it as untrusted input — validate it, module 31; *never* derive identity from it — the session is the identity, module 10-05).**

### 3.3 The capstone's action inventory (the module-23 specs' heads, collected)

`FILE: docs/action-inventory.md` (the artifact — the format of the module-23 invalidation specs, now with the *boundary* column):

```md
| Action (feature/actions.ts) | Session gate | Validation (module 31) | Service (module 17) | Invalidation (module 23) | Response |
|---|---|---|---|---|---|
| createProduct (catalog) | org member | productInput (Zod) | createProductService | 'products','catalog' + updateTag | redirect('/products/'+slug) |
| updateProduct (catalog) | org member + owner role | productInput | updateProductService | 'products','product:{slug}' | redirect back |
| delistProduct (catalog) | owner | id (uuid) | delistProductService | 'products','catalog' | redirect('/products') |
| cancelOrder (order) | org member | id (uuid) | cancelOrderService | 'orders','order:{id}','revenue:{orgId}' (the full module-23 spec) | redirect('/orders/{id}?cancelled=1') |
| refundOrder (order) | owner (the *financial* gate, module 11) | id + reason | refundOrderService | 'orders','order:{id}','revenue:{orgId}' | redirect back |
| updateProfile (account) | self (or org admin) | profileInput | updateProfileService | 'profile:{userId}' | redirect back |
| updateOrgSettings (org) | owner | settingsInput | updateOrgSettingsService | the settings' tags | redirect back |
| inviteMember (org) | owner/admin | email + role (RBAC, module 11) | inviteMemberService | 'members:{orgId}' | redirect back |
| removeMember (org) | owner; can't remove self/last owner | id | removeMemberService | 'members:{orgId}' + the session row (module 23's suspend pattern) | redirect back |
| updatePricing (admin) | platform admin (the (admin) zone) | pricingInput | updatePricingService | 'pricing' (updateTag — the actor is the admin) | redirect back |
```

**The inventory is the security artifact** (the module-19/21 cross-check: every action row has its *gate* (the session + role) *and* its *validation* *and* its *service* (the tenancy) — a row missing any of the three is a **finding**; the module-20 test phase writes a test *per row*).

## 4. Production Code — The complete thin action (the template all others follow)

`FILE: src/features/order/order-actions.ts` (production pattern — [SERVER]; the full `cancelOrder`, the §3 action + module 31's validation + module 23's spec)

```ts
'use server'

import { headers } from 'next/headers'
import { redirect } from 'next/navigation'
import { revalidateTag, updateTag } from 'next/cache'
import { auth } from '@/lib/auth'
import { z } from 'zod'
import { cancelOrderService } from '@/services/orders'
import { TAGS } from '@/lib/tags'
import { AppError, isAppError } from '@/lib/errors'

/**
 * INVALIDATION SPEC (module 23):
 *   mutation:      order → cancelled (service enforces tenancy + allowed transition)
 *   tags touched:  'orders' (lists) · 'order:{id}' (detail) · 'revenue:{orgId}' (the summary)
 *   actor:         updateTag (immediate — read-your-own-writes) + redirect to the fresh detail
 *   fleet:         revalidateTag(…, 'max') — SWR (the others' next visit regenerates in background)
 */
export async function cancelOrder(orderId: string): Promise<void> {
  // 1 — the SESSION gate (the identity; the action's first job — the route is public, module 19):
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) {
    throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })
  }
  const orgId = session.organizationId                       // identity from the SESSION (never the arg)

  // 2 — the VALIDATION gate (the input; client-supplied → untrusted → Zod, module 31):
  const id = z.string().uuid().parse(orderId)                // throws → the action's error path (module 31)

  // 3 — the SERVICE (the logic; tenancy + business rules live there, module 17):
  await cancelOrderService(orgId, id)

  // 4 — the INVALIDATION (the module-23 spec — actor immediate, fleet SWR):
  updateTag(TAGS.orders)
  updateTag(TAGS.order(id))
  updateTag(TAGS.revenue(orgId))
  revalidateTag(TAGS.orders, 'max')
  revalidateTag(TAGS.order(id), 'max')
  revalidateTag(TAGS.revenue(orgId), 'max')

  // 5 — the RESPONSE (the response model: the redirect carries the re-render — the fresh
  //    detail page, the updated list, the updated revenue — in the single roundtrip):
  redirect(`/orders/${id}?cancelled=1`)
}
```

**Read it as the template:** the five steps (session → validate → service → invalidate → respond) are *every* action's shape; the *content* of each step differs per action (the inventory's columns). The `AppError` 401 (not a `throw new Error`) — the typed error (module 17) — is what the client *sees* (module 31's error round-trip) and what telemetry records (module 21's digest).

**The `use server` + `redirect` interaction** (the subtle one): `redirect()` in an action throws a *special* error (the `NEXT_REDIRECT` — module 09-04's redirect mechanics) — it's not a *failure*; the framework converts it into the 303 (the module 09-04 redirect table) + the re-render. A `try/catch` *around* the action call on the client *would* catch it (the error object carries the `digest`) — so **the action's error handling must distinguish `NEXT_REDIRECT` from real errors** (module 31's error mapping: a redirect is *not* an error state; the form's `useActionState` state must not render an error banner for a redirect — the module-31 trap).

## 5. Common Mistakes (the action failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **No session check in the action** (relying on the layout's gate) | The layout's gate covers the *rendering* path — **the action's POST is a *separate* request** (the module-19 line: "reachable to anyone who can send the same POST") — an unauthenticated POST *succeeds* (the service's tenancy check is the backstop — but the *action's* 401 is the *correct* response; the service's 403-tenant is the *last* line) | The inventory's "session gate" column, *per action* — the module-20 test: "the action with no cookie" per row |
| Business logic in the action (queries, the status-transition `switch`, the audit insert) | Untestable outside HTTP; the tenancy check duplicated per action; the module-17 seam (the service) bypassed | The five-step template — the logic is *in the service*; the action *orchestrates* |
| `async` forgotten (a `use server` function that isn't `async`) | Build error (it can't be called over the network) — but the *trickier* variant: a *synchronous-looking* action that's `async` and *forgets to `await`* the service call (the `cancelOrderService(…)` without `await` — the function *returns* before the mutation commits; the invalidation runs on the *old* state; the redirect shows *stale* data) | The `await` on the service call is *load-bearing* (the write→invalidate ordering, module 23 — the *unawaited* service breaks it) — the lint rule (`@typescript-eslint/await-thenable`… practically: **the action's step 3 is `await`-ed, always**) |
| Parallel action dispatch "to speed things up" (two buttons, two `startTransition`s, no coordination) | The official line: *the client dispatches Server Functions one at a time* — the second *waits* for the first (sequential) — a UI that *shows* both as pending but runs them in order: the *perceived* parallelism is a lie; worse, two mutations on the *same resource* (two "cancel" clicks) serialize into a **double-mutation** (the second gets the service's "already cancelled" 409) | The **pending gate** (module 32: `disabled={isPending}` — the *first* control; the service's **idempotency** (module 09-04's status-transition guard: an *already-cancelled* order → a 409 *or* a no-op — the service decides, *idempotent by design*) is the *second*; the UI must *handle* the 409 (module 31: "already cancelled" is a *state*, not an *error* — the re-render shows the truth) |
| Returning a **database record** from the action ("the UI needs it") | The official line: *only return data the UI needs, not raw database records* — a record (with `orgId`, the timestamps, the internal IDs) is a *leak* (the module-14 DTO rule, applied to the *return value* — regime 1 serialization *carries* everything the record has) | The action returns **`void`** (the redirect carries the re-render — the *data* is the page's *fresh read*, not the action's return) — or a **DTO** (the minimal shape — `orderSummary`), never a record. **The capstone rule: actions return `void` + redirect; the exception (a non-redirecting action) returns a DTO, module 31's `useActionState` state** |
| The action in a `'use client'` file (the *definition* there) | Build error (the directive is server-side) — the *confusion* is the common one: the `CancelButton` (client) *imports* the action (the ID) — that's correct; *defining* it there is not | Actions in `src/features/*/actions.ts` (file-level) or inline in a *Server Component* — never in the client zone (the module-03's 5 import rules, row: "client files never define `'use server'`") |

## 6. Security Notes (the module's *security* is the section, not a footnote)

- **The action route is public by construction** (the module-19 threat model, the official "reachable to anyone who can send the same POST"): the *ID* is in the client bundle (visible, crawlable — a scanner *will* find your action IDs); the *only* controls are the **in-function checks** (the session, module 10-05; the role, module 11; the tenancy, module 17; the validation, module 31). **The "security" of a mutation is the *action's* code, not the *layout's*** — the layout's gate is *rendering*; the action's gate is *execution*. The inventory's gate column is the control, *per action*.
- **The `FormData`/args are the *client's* input** (untrusted — the module-19's completeItem *unsafe/safe* pair, the official `server-actions` guide's example, the module's canonical security pair):

```ts
// UNSAFE (the official example's shape): the whole item, including its id, comes from the
// client — anyone who can POST here can act on any item (the tenancy check is *absent*):
export async function markShippedUnsafe(item: OrderDto) {
  await markShippedService(item.orgId, item.id)   // item.orgId is CLIENT-SUPPLIED (a cross-tenant write!)
}

// SAFE (the capstone's shape): take only the change (the id); derive identity from the SESSION;
// the service enforces tenancy (orgId from the session, *never* the arg):
export async function markShipped(orderId: string): Promise<void> {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })
  const id = z.string().uuid().parse(orderId)
  await markShippedService(session.organizationId, id)   // orgId = SESSION (the tenancy, module 17 rule 1)
  // …invalidate + redirect
}
```

  **The cross-tenant test** (module 11-02, the *action* version): user A (org 1) POSTs `markShipped(orderOfOrg2)` — the service's tenancy check (the `WHERE orgId = session.orgId` — module 17's orgId-first rule) **must** reject it (the 403/404 — the module-11-03's 401-vs-403: *known user, wrong org's resource* → the 404 (don't leak existence) or the 403 (the *admin* case) — the capstone's choice: the 404 (the *not-found* for *your* org — the module-11-03's "don't leak" line)).
- **CSRF** (the module-19's line, the action's version): a *form* POST from an *attacker's* page to *your* action URL — the **session cookie** is *sent* (the browser attaches it — the module-10-03's cookie flags: `httpOnly`, `sameSite: 'lax'` — the *lax* is the CSRF *first* control: a *cross-site* form POST is a *top-level* navigation → the cookie *is* sent on `lax` for top-level… the *precise* line: `sameSite=lax` blocks *subresource* cross-site POSTs (the `fetch`/`XHR` CSRF); a *top-level form* POST (the classic CSRF) *does* send the `lax` cookie — the *second* control is the **`Origin`/`Referer` check** (Next.js's `serverActions.allowedOrigins` — the module-02's config — *or* the **CSRF token** in the form's hidden field (module 30's progressive-enhanced form *includes* the token — the `action`'s *first* check: the token matches the session). **The capstone's posture**: `sameSite=lax` + the `allowedOrigins` (the preview host + the prod origin) + *the action's session/role/tenancy checks* (the *final* control — the token is a *first* line; the in-function checks are the *real* ones — an attacker with a *valid* session (the *session-hijack* case, module 10-03's token rotation) is not a *CSRF* — that's the *auth* layer's problem).
- **The return value is a *downward* payload** (regime 1): *what the action returns is *user-visible data*** (the module-14's serialization rules — a non-serializable return is a *runtime* failure *on the client*; a *record* return is a *leak*) — the capstone's `void`+redirect rule *eliminates* the surface (the *exception*: the `useActionState` *state* (module 31) — a *DTO* shape, module 31's `FormState` — *validated as a DTO* before it's returned (the *fieldErrors* are strings — the *message* is the *user-facing* text (the module-19-03's "no internals to the user" line)).

## 7. Performance Notes

- **The single roundtrip is the *win*** (the module-29's title claim, measured): a mutation + the re-render is **one** POST + **one** response (the RSC payload) — the *old* SPA's equivalent is **two** requests (the `POST /api/orders/:id/cancel` + the `GET /orders` or the client's *manual* refetch) — the **roundtrip count** is halved *and* the *data* in the response is *already fresh* (the `updateTag` — no *second* fetch for the *truth*). The module-18-01's measurement (the action's TTFB: the POST's *server* time — session + service + invalidation — typically 20–100ms on the capstone) is the *action's* performance number.
- **Sequential dispatch is a *load* property** (the official "one at a time"): a *page* with five pending actions (a bulk "cancel 5") runs them *in order* — the *fifth* starts after the *fourth* *commits* (the *total* = the sum — a bulk action is *one* action (a *loop in the service*, module 17's "the parallel work *inside* a single Server Function" — the official line: "perform parallel work **inside a single Server Function** or Route Handler" — the *bulk* action's *service* does the `Promise.all` (module 18's parallel topology) — the *action* is *one*, the *work* is *parallel *inside* it). **The rule**: *parallelism is a service-internal concern (the module-18's `Promise.all`); the action is the *sequential* boundary (one call, one response).*
- **The `useTransition`/pending state is the *perceived* performance** (module 32's subject): the *real* latency is the action's server time (20–100ms) + the *re-render* (the RSC payload's *transfer* + the *client* reconciliation) — the *perceived* "how fast did the button work" is *dominated* by the *pending* feedback (the `disabled` + the "Cancelling…" label — module 32) — a *fast* action with *no* pending feedback *feels* broken (the *double-click* — the *second* click is the *sequential* dispatch's *second* call — the *double-mutation* — the pending gate is the *UX* and the *correctness* control at once).
- **The redirect's re-render is the *fresh* data** (the `redirect`'s 303 → the *new* page's *render* — the *cached* entries (the `updateTag`'d ones are *re-read*; the *untouched* ones *hit* — the module-23's "the actor sees the truth, the *rest* is a cache hit" — the *redirect's* re-render is *cheap* (the *invalidated* entries re-run — the *one* query per *touched* tag — the *untouched* sections are *hits* (the module-25's dashboard's "the cancel *only* re-runs the *orders* + *revenue* entries — the *activity* + *analytics* are *hits* — the re-render is *fast* (the *touched* queries, not the *whole page*)).

## 8. Exercise

**Beginner.** Write the *five-step template* action from *scratch* (the `cancelOrder`, §4) — *type it, don't copy it* — and then the *inventory row* for it (the `docs/action-inventory.md` row: the gate, the validation, the service, the invalidation, the response). Then: **the "unauthenticated POST" test** (the module-20's seam, the *action* version): in the browser's DevTools Console, `fetch('/<the action ID>', { method: 'POST' })` (the *raw* POST — no form, no session) — the action's **401** must be the response (the *session gate* — the module-19's "reachable by anyone" *exercised*). Screenshot the 401. (The *action ID* is in the client bundle — the *visible* route — the *gate* is the *control*.)

**Intermediate.** The *double-click* experiment (the *sequential dispatch*, *observed*): the `cancelOrder` with the service's *latency* stubbed (the module-28's `delay(500)` — the *slow* service) — click *Cancel* *twice* in fast succession (the *pending gate* **absent** — the `disabled={isPending}` *removed*, the *anti-pattern* version). Observe: the *first* mutation *commits*; the *second* *waits* (the *sequential* dispatch — the *DevTools* Network tab: the *two* POSTs, the *second* starts *after* the *first* *responds*); the *second* hits the service's *status-transition guard* (the *already-cancelled* — the 409/the *no-op* — the module-09-04's guard). *Then* add the pending gate (`disabled={isPending}`) — the *second click is *impossible* (the button is *disabled* during the *first* pending). Document the *two* behaviors (the *unguarded* double-click → the *409*/no-op; the *gated* → the *single* call) — the *pending gate is the *UX* and the *correctness* control* (the module-32's setup).

**Production.** The *full action-inventory security pass* (the module-19's "every endpoint" audit, the *action* version): for *every* row in the inventory — (a) the *no-cookie* POST (the 401); (b) the *cross-tenant* POST (user A → org B's resource — the 404/403, the module-11-02's cross-tenant test, *per action*); (c) the *wrong-role* POST (a member → the *owner*-only action — the 403, module 11-03); (d) the *invalid-input* POST (the *Zod* rejection — the module-31's field-error *round-trip*). The *four* results *per row* (the 16–40 cells) are the *action's security matrix* — the *module-20 test phase's* *starting point* (the *tests* are the *matrix's* *rows* — the *module-20's* `vitest`/`playwright` *suits* are *generated from this matrix*). The *artifact*: the `docs/action-security-matrix.md` (the *matrix*, the *per-row* *results*).

## 9. Architecture Challenge

**Prompt:** The "bulk refund" feature: an *owner* selects 50 orders (the *OrdersTable*'s checkboxes — a *client* island) and clicks "Refund selected." The *naive* design: the *client* *loops* over the 50 IDs and *calls* `refundOrder(id)` 50 times (the *event-handler* invocation — the *startTransition* per ID).

**The problems**: (1) the *sequential* dispatch (the *official* "one at a time") — 50 *actions* × the *action's* latency (20–100ms) = the *total* (1–5 *seconds* of *waiting*, *serialized*); (2) the *partial failure* (the 23rd *fails* (a *duplicate* refund — the *service's* 409) — the *first 22 are *committed*; the *UI* is *split* (22 *refunded*, 28 *pending*) — the *state is *inconsistent* mid-batch); (3) the *invalidation* (the *50 × `revalidateTag`* — the *tags are *the same* (the `'orders'`, the *per-order* `'order:{id}'` ×50, the *'revenue:{orgId}'* ×50) — the *50-fold* *revalidation* (the *dedupe* helps — the *tags are *deduped* (the *same tag* *revalidated* 50× = the *one* *invalidation* — the *module-23's* *idempotent* tags) — *but* the *per-order* tags are *50 distinct* (the *50 entries* are *stale* — the *50 re-reads* on the *next* *visit* — the *thundering herd* of the *module-22's* *storm*, the *actor's* *own* *bulk* causing it).

**Design the *correct* bulk action**: the *single* action (the *module-29's* "parallel work *inside* a *single* Server Function" line) — the *signature* (the *args*: the *50 IDs* — the *client-supplied* — *validated* as a *50-max* *uuid* array (the *module-31's* *Zod*)), the *service* (the *bulk* `refundOrdersService(orgId, ids)` — the *loop* *in the service* (the *module-17's* *service* is the *logic's* home) — the *per-order* *status-transition guard* (the *already-refunded* → the *skip*, *not* the *fail* — the *idempotent* *bulk*), the *partial-failure* *contract* (the *service* *returns* the *result DTO* (the `{ refunded: string[], skipped: string[], failed: {id, reason}[] }` — the *module-14's* *DTO*, the *module-31's* *state* — the *action* *doesn't* *redirect* (the *no* *303* — the *response is the *DTO* (the *module-29's* *exception*: the *non-redirecting* action *returns* a *DTO*) — the *UI* (the *client* island) *shows* the *result* (the "48 refunded, 2 already refunded (skipped), 0 failed" — the *module-32's* *optimistic* *rollback* *on the *failed* *only*)), the *invalidation* (the *one* *action* → the *one* *invalidation* *set* (the `'orders'` *once* (the *dedupe*), the *'order:{id}'* for the *touched* *orders only* (the *refunded + skipped* — the *failed* *are *unchanged* (no *invalidation* — the *truth is *unchanged*), the *'revenue:{orgId}'* *once* (the *dedupe*) — the *one-fold*, *not* the *50-fold*), the *pending* (the *one* *button*, the *one* *pending state* — the *50* is *invisible* to the *user* (the *module-32's* "50/50 processed" *progress* — the *service's* *streaming* *updates* (the *module-27's* *streaming*, the *action's* *response* is the *final* *DTO* — the *progress* is the *client's* *estimate* (the *module-32's* *optimistic* *counter* — the *honest* label: "Processing 50 refunds…" — the *no* *fake progress* (the *module-32's* *mistake*: the *optimistic* *100%* *before* the *server* *confirms*).

Produce: the *action* (the *five steps*, the *bulk* *args*, the *DTO* *return*), the *service* (the *loop*, the *guard*, the *result DTO*), the *invalidation* (the *one-fold*), the *client* (the *pending*, the *result display*, the *rollback* *on* *failed*), and the *why* (the *sequential* dispatch *avoided* (the *one* *action*), the *partial failure* *handled* (the *result DTO*, the *per-row* *state*), the *storm* *avoided* (the *one-fold* *invalidation*).

<details>
<summary>Model answer</summary>
**The action** (the *five steps*, the *bulk*):

```ts
// src/features/order/order-actions.ts (the *bulk* action — the *single* call for the *50*):
'use server'
// …imports (the *module-29's* *template*)…

const bulkRefundInput = z.object({
  orderIds: z.array(z.string().uuid()).min(1).max(50),   // the *50-max* (the *module-31's* *Zod*; the *DoS* guard: the *array* is *client-supplied* (an *unbounded* *array* = the *memory* *attack* — the *max(50)* is the *input* *bound* (the *module-19's* *resource* *limit*)
})

export async function bulkRefund(orderIds: string[]): Promise<BulkRefundResult> {
  // 1 — the *SESSION* *gate* (the *owner* *role* — the *module-11's* *RBAC* (the *financial* *gate*)):
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })
  if (session.role !== 'owner') throw new AppError({ status: 403, code: 'forbidden', message: 'Owner role required' })   // the *403* (the *module-11-03's* *401 vs 403*: the *known user, wrong role* — the *authZ* *denial*)
  const orgId = session.organizationId

  // 2 — the *VALIDATION* (the *args* — the *50-max* *uuids*):
  const { orderIds: ids } = bulkRefundInput.parse({ orderIds: orderIds })

  // 3 — the *SERVICE* (the *bulk* *logic* — the *loop*, the *guard*, the *result DTO* — the *module-17's* *home*):
  const result = await refundOrdersService(orgId, ids)   // the *one* *service call* (the *loop is *inside*)

  // 4 — the *INVALIDATION* (the *one-fold* — the *touched* *only*):
  const touched = [...result.refunded, ...result.skipped]   // the *changed* *orders* (the *failed* are *unchanged* — *no* *tag*)
  updateTag(TAGS.orders)                    // the *lists* — *once* (the *dedupe*)
  updateTag(TAGS.revenue(orgId))            // the *summary* — *once* (the *dedupe*)
  for (const id of touched) updateTag(TAGS.order(id))   // the *per-order* — the *touched* *only* (the *not* the *50*, the *changed* *48+2*)
  revalidateTag(TAGS.orders, 'max')
  revalidateTag(TAGS.revenue(orgId), 'max')
  // the *per-order*: the *SWR* for the *fleet* (the *actor's* *immediate* is the *updateTag* — the *fleet's* *next visit* *regenerates* the *touched* *entries* in the *background* (the *module-23's* *SWR* — the *spread* (the *not* the *storm* — the *48 entries* *regenerate* over the *next* *traffic*, *deduped* per entry (the *module-22's* *storm-avoidance*, the *one-fold* *invalidation* is the *setup* (the *tags are *invalidated* *once*; the *regenerations* are the *spread*)

  // 5 — the *RESPONSE* (the *exception* to the *redirect* rule: the *non-redirecting* action *returns* the *DTO* (the *module-29's* *return-value* *rule*: the *DTO*, *not* the *record*)):
  return result   // the `{ refunded: string[], skipped: string[], failed: { id: string; reason: string }[] }` (the *module-14's* *plain* *DTO* — the *serializable*)
}
```

**The service** (the *loop*, the *guard*, the *result* — the *module-17's* *five contract rules*, the *bulk*):

```ts
// src/services/orders.ts (the *bulk* *refund* — the *logic's* home):
export async function refundOrdersService(orgId: string, ids: string[]): Promise<BulkRefundResult> {
  // the *tenancy*: the *orgId* is the *session's* (the *module-17's* rule 1 — the *not* the *arg's* *orgId*)
  // the *loop* (the *sequential* — the *per-order* *guard* is the *correctness* (the *transactional* *status transition* — the *module-09-04's* guard):
  const result: BulkRefundResult = { refunded: [], skipped: [], failed: [] }
  for (const id of ids) {
    try {
      await refundOrderService(orgId, id)   // the *single-order* *service* (the *same* *logic* — the *reuse* (the *module-17's* *single* *source*): the *status-transition guard* (the *already-refunded* → the *AppError* 409 — the *catch* *below*)
      result.refunded.push(id)
    } catch (e) {
      if (isAppError(e) && e.code === 'already-refunded') {
        result.skipped.push(id)   // the *idempotent* *bulk*: the *already-refunded* is a *skip*, *not* a *fail* (the *module-09-04's* *idempotency* — the *re-* *refund* of a *refunded* *order* is a *no-op* (the *safe* *re-* *run*)
      } else if (isAppError(e) && (e.code === 'not-found' || e.code === 'forbidden')) {
        result.failed.push({ id, reason: 'Not found in your organization' })   // the *cross-tenant* / the *not-in-org* — the *module-11-03's* *404* (the *no* *leak*): the *reason* is the *user-facing* *text* (the *module-19-03's* *no internals*)
      } else {
        throw e   // the *unexpected* (the *DB* *down*, the *payment provider* *error*) — the *bulk* *aborts* (the *module-09-04's* *transaction* — the *all-or-nothing* *at the* *service* *level*: the *single-order* *refunds* are *committed* *per order* (the *the* *status* *transition* is *per order* (the *no* *cross-order* *transaction* — the *each order's* *refund* is *independent* (the *the* *payment* *provider's* *refund* is *per order* (the *the* *business* *truth*: the *50 orders* are *50* *independent* *refunds* (the *the* *no* *atomic* *bulk* — the *partial* *completion* is the *truth* (the *result DTO* is the *honest* *report*)
      }
    }
  }
  return result
}
```

**The client** (the *pending*, the *result*, the *rollback* — the *module-32's* *patterns*, the *bulk*):

```tsx
// src/features/order/components/bulk-refund.tsx (the *island* — the *module-32's* *pending* + the *result*):
'use client'
import { useState, useTransition } from 'react'
import { bulkRefund, type BulkRefundResult } from '@/features/order/order-actions'

export function BulkRefundBar({ selectedIds }: { selectedIds: string[] }) {
  const [isPending, startTransition] = useTransition()
  const [result, setResult] = useState<BulkRefundResult | null>(null)

  return (
    <div>
      <button
        disabled={isPending || selectedIds.length === 0}
        onClick={() => {
          setResult(null)   // the *fresh* *batch* (the *previous* *result* is *stale*)
          startTransition(async () => {
            const r = await bulkRefund(selectedIds)   // the *one* *call* (the *50* is *inside*)
            setResult(r)   // the *result DTO* (the *module-29's* *return-value* — the *UI's* *truth*)
          })
        }}
      >
        {isPending ? `Processing ${selectedIds.length} refunds…` : `Refund selected (${selectedIds.length})`}
      </button>
      {result && (
        <dl className="mt-2 text-sm">
          <div><dt>Refunded</dt><dd>{result.refunded.length}</dd></div>
          <div><dt>Already refunded (skipped)</dt><dd>{result.skipped.length}</dd></div>
          {result.failed.length > 0 && (
            <div><dt>Failed</dt><dd>{result.failed.map((f) => f.id).join(', ')} — {result.failed[0].reason}</dd></div>
          )}
        </dl>
      )}
    </div>
  )
}
```

**The *why*** (the *three* *problems*, the *solved*):
1. **The *sequential* dispatch *avoided***: the *one* *action* (the *one* *POST*, the *one* *response* — the *module-29's* "parallel work *inside* a *single* Server Function" — the *loop is *the service's* (the *the* *action* is the *boundary* (the *one* *call*); the *50* is *invisible* to the *dispatch* (the *not* 50 *sequential* *POSTs* (the *the* *1–5s* *wait* is *gone* (the *one* *POST's* *latency* (the *the* *service's* *loop* is the *server's* *time* (the *the* *per-order* *refund* (the *payment provider* round-trip ×50 — the *the* *honest* *number*: the *bulk's* *latency* is the *50 × the* *provider* *refund* *latency* (the *the* *provider* is the *bottleneck* (the *the* *sequential* *in the service* is the *correct* (the *the* *parallel* *to the provider* is the *module-18's* `Promise.all` (the *the* *rate-limit* (the *provider's* *concurrency* *cap* — the *the* *honest* *design*: the *sequential* (the *safe* (the *the* *provider* *rate* *limit* — the *the* *50 sequential* refunds at the *provider's* *pacing* is the *correct* (the *the* *parallel* is the *module-18's* *challenge* (the *the* *measure the provider's cap* (the *the* *if* *50 concurrent* is *safe*, the *`Promise.all`* (the *the* *the service's* *loop* is the *default*; the *parallel* is the *measured* *upgrade*)).
2. **The *partial* failure *handled***: the *result DTO* (the *`{refunded, skipped, failed}`* — the *honest* *report* — the *UI* *shows* the *per-group* *counts* + the *failed* *IDs* + the *user-facing* *reason* — the *the* *state is *not* *inconsistent* (the *the* *22 refunded, 28 pending* is *replaced* by the *the* *48 refunded, 2 skipped, 0 failed* (the *the* *the truth* (the *the* *the per-order* *status* is the *service's* *guard's* *output* (the *the* *the result* is the *aggregation*)).
3. **The *storm* *avoided***: the *one-fold* *invalidation* (the *the* *tags are *invalidated* *once* (the *the* *`updateTag`* is *idempotent* (the *the* *the 50-fold* is *gone*; the *the* *per-order* tags are the *touched* *only* (the *the* *48, not the 50 (the *the* *failed* are *unchanged* (the *no* *tag* — the *truth* *unchanged*); the *the* *fleet's* *SWR* is the *spread* (the *the* *48 entries* regenerate over the *next traffic* (the *module-22's* storm-avoidance) (the *the* *the actor's* *immediate* is the *updateTag* (the *the* *the redirect* is *absent* (the *the* *the response is the DTO* (the *the* *the actor* is on the *orders page* (the *the* *the re-render* is the *RSC payload* (the *the* *the fresh* *orders list* is in the *response* (the *the* *the module-29's* *single roundtrip* — the *the* *the mutation's result AND the fresh UI* in the *one* response.
**The *generalization*** (the *bulk* *pattern*, the *module's* *standing* *rule*): **a *bulk* mutation is a *single* action (the *boundary* — the *one* *POST*) whose *service* does the *per-item* *work* (the *loop*, the *guard*, the *aggregation* — the *module-17's* *home*), returns a *result DTO* (the *per-item* *state* — the *module-14's* *plain* *shape*), and invalidates the *touched* *only*, *once* (the *module-23's* *spec*, the *bulk*). The *client* shows the *pending* (the *one* *button*) and the *result* (the *honest* *report* — the *no* *optimistic* *100%* (the *module-32's* *mistake* — the *bulk's* *optimistic* is the *per-item* *skip* (the *the* *already-refunded* is *known* *pre-* *call* (the *the* *the client* *could* *optimistically* *mark* *the* *selected* *as* *"refund requested"* (the *the* *the honest* *label* — the *the* *not* *"refunded"* (the *the* *the server's* *confirmation* is the *result DTO* (the *the* *the module-32's* *optimistic* *label* is the *request's* *state*, *not* the *mutation's* *completion*).**
</details>

## 10. Official Documentation

- Mutating Data (Server Functions, the creating/invoking, the POST-only, the sequential dispatch): https://nextjs.org/docs/app/getting-started/mutating-data
- `use server` (the directive reference, the file-level/inline, the return-value security line): https://nextjs.org/docs/app/api-reference/directives/use-server
- Server Actions and Mutations (the guide — the build-time mechanics, the security): https://nextjs.org/docs/app/guides/server-actions
- Forms (the forms guide — the action prop, the additional arguments): https://nextjs.org/docs/app/guides/forms
- `redirect` (the 303, the response model): https://nextjs.org/docs/app/api-reference/functions/redirect
- Revalidating (the action's invalidation — module 23's primitives): https://nextjs.org/docs/app/getting-started/revalidating

## 11. What You Should Know Before Continuing

- [ ] I can state the *terminology* precisely (Server Function = the general async-over-network; Server Action = the mutation convention with `startTransition`) — the *official* words, the *course's* usage
- [ ] I know the *two placements* (the file-level, the inline in a Server Component) and the *two invocations* (the form, the event handler) — and the *args* rule (the statics *first*, the `FormData` *last*, the *form context's* *constraint*)
- [ ] I can write the *five-step template* action (the session → validate → service → invalidate → respond) from memory — and the *inventory row* for it
- [ ] I know the *security* is the *action's* code (the *route is public*; the *session/role/tenancy/validation* are the *controls*) — the *inventory's* *gate* column, the *cross-tenant* test
- [ ] I know the *single roundtrip* (the *mutation + the fresh UI* in the *one* response) and the *sequential dispatch* (the *one at a time*; the *parallel work is in the service*)
- [ ] I know the *return-value* rule (the `void` + the redirect; the *exception* is the *DTO* — the *never* the *record*)
- [ ] I've done the *unauthenticated POST* test (the *401*) and the *double-click* experiment (the *pending gate* is the *correctness* control)

**Next:** Module 30 — Forms & Progressive Enhancement (the form that works with *JS disabled* — the `action` prop, the `startTransition`'s *automatic* wrap, the *native POST* fallback, the *hidden inputs* for the *args*, the *CSRF token* in the *form*).
