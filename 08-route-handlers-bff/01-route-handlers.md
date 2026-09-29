# Module 34 — Route Handlers Deep Dive: The Real HTTP Endpoints

**Phase 8: Route Handlers & BFF · Module 34 of 101**

> **Where does this run?** `[SERVER]` — a Route Handler is a **plain HTTP endpoint** (the Web `Request` in, a `Response` out), living at `app/**/route.ts`. It is *not* a page (no HTML for humans), *not* an action (no RSC payload, no session assumption) — it's the **machine's door** (module 33's Q1). The module-23 webhook and the module-33 partner API both live here; this module makes the *whole* surface deliberate.

---

## 1. Concept — What a Route Handler is (and the 4 things it is not)

**The file convention** (module 05's table, this row): `app/api/orders/route.ts` → `POST/GET /api/orders`. The export **is** the HTTP method handler:

```ts
// FILE: src/app/api/orders/route.ts — [SERVER]
import { NextResponse } from 'next/server'
import { z } from 'zod'
import { listOrdersService } from '@/services/orders'      // module 17: the SAME service the actions use (module 33's invariant)
import { getApiKeyOrgId } from '@/lib/api-auth'             // module 36: the machine's auth (API key)

const querySchema = z.object({
  orgId: z.string().uuid(),
  status: z.enum(['pending', 'paid', 'shipped', 'cancelled']).optional(),
  cursor: z.string().optional(),
  limit: z.coerce.number().min(1).max(100).default(20),
})

export async function GET(request: Request) {
  // 1 — AUTH (the machine's: the API key — module 36; the SESSION is NOT assumed here):
  const orgId = await getApiKeyOrgId(request)
  if (!orgId) return NextResponse.json(ApiErrorBody(401, 'unauthorized', 'Missing or invalid API key'), { status: 401 })

  // 2 — VALIDATE (the query string is CLIENT-SUPPLIED input — module 31's Zod, module 04's truth):
  const parsed = querySchema.safeParse(Object.fromEntries(request.nextUrl.searchParams))
  if (!parsed.success) return NextResponse.json(ApiErrorBody(422, 'validation', 'Invalid query', parsed.error.issues), { status: 422 })

  // 3 — SERVICE (module 17's tenancy: orgId from the KEY (not the URL — module 33's "derive, don't take")):
  const { status, cursor, limit } = parsed.data
  const page = await listOrdersService(orgId, { status, cursor, limit })

  // 4 — RESPONSE (module 36's contract: the envelope, the pagination headers):
  return NextResponse.json(
    { data: page.items, meta: { nextCursor: page.nextCursor } },
    { headers: { 'Cache-Control': 'no-store' } },
  )
}
```

**The 4 things it is not** (the module-33 classifier, enforced):

| It is NOT | The tell | The correct home |
|---|---|---|
| A **page** | `return new Response('<html>…')` / an HTML string | `app/**/page.tsx` (module 05's routes — the human's door, module 33's Q1) |
| An **action** | A client island `fetch('/api/…')` to drive the app's UI | The `'use server'` action (module 29 — the RSC re-render is the action's exclusive, module 33's Q2) — the module-16 self-API anti-pattern |
| A **proxy to its own app** | `fetch(internalUrl)` re-exposing the same data with no reshaping | The service (module 17) — the module-16 deletion checklist |
| A **database function** | Drizzle/SQL *directly* in the route | The service (module 17's rule: the route calls the service — the module-33 invariant; the route is the *transport*, the service is the *logic*) |

**The Web APIs are the contract** (verified current behavior): the handler receives the standard **`Request`** (the Web API — module 01's stack note: `NextRequest`, the `next/server` subclass with `cookies`/`nextUrl` conveniences, is *available*; the capstone's rule: **standard `Request` first** (portable — the module-22-01's "same code runs on Vercel or Docker" line), `NextRequest` only when its conveniences are *needed* (the `cookies()` helper, module 34 §5 — the module-02's verified stack: both exist in 16.3.x)). The response: the standard **`Response`** (or `NextResponse` — the `json()`/`redirect()` conveniences). **The handler is an async function returning a `Response`** — nothing more (everything else is built from this).

## 2. Mental Model — The 5-method surface (what each verb *means* here)

| Method | The semantics (the module-19's "no GET mutation" line, module 33's mistake #7) | The capstone's uses | Cacheability (module 20's L2) |
|---|---|---|---|
| `GET` | **Read** — *idempotent*, *safe* (the browser/CDN may cache it — the module-20's L2) | `GET /api/v1/products` (the public catalog, module 36), `GET /api/v1/orgs/:orgId/orders`, `GET /api/health` (module 33's #9), `GET /api/descriptions/:id/stream` (the SSE, module 33's challenge) | **Yes** (the `Cache-Control` decides — the module-36's per-endpoint policy; the *public* catalog: `s-maxage=60, stale-while-revalidate=300` (the CDN — module 20's L2); the *org-scoped*: `no-store` (the API key in the header ≠ a URL cache key — the module-36's line: **key-authenticated GETs are `no-store` by default** (the CDN can't vary by header without config — the *honest* default: no-store) — the module-36's line: the *public* GETs are CDN-cacheable; the *keyed* GETs are not (the header-vary problem)) |
| `POST` | **Create / command** — *not* idempotent by default (the module-09-04's idempotency is the *service's* job, module 33's line) | `POST /api/v1/orgs/:orgId/orders/:id/cancel` (the module-33's door 2), `POST /api/webhooks/stripe` (the module-23's §3.3 — the 2xx-fast rule), `POST /api/v1/feedback` (the public form's *machine* fallback — module 33's #3's *note*: the public form is an *action* (the module-30's floor) — the POST is the *partner's* feedback endpoint) | No (the module-20's L2: the POST is never cached) |
| `PUT` | **Full replace** — *idempotent* (the same PUT twice = the same state — the module-19's line) | `PUT /api/v1/orgs/:orgId/settings` (the partner's full settings replace — the module-36's contract: the *body is the complete state*) | No |
| `PATCH` | **Partial update** — *not* idempotent by default (the `+1` semantics) | `PATCH /api/v1/orgs/:orgId/products/:id` (the partner's field update — the module-36's contract: the *body is the delta*, the module-31's Zod per-field) | No |
| `DELETE` | **Remove** — *idempotent* (the second DELETE = the 404, the module-11-03's "don't leak" line: the *soft-delete* (module 30's delist) makes the second DELETE a *no-op 404* — the *module-19's line: the DELETE's idempotency is the service's* (module 09-04's guard)) | `DELETE /api/v1/orgs/:orgId/products/:id` (the partner's delist — the *soft-delete* (module 30's) — the *no hard delete via the API* (the module-19's line: the *destructive* is the *admin's* (module 11's RBAC) — the *API's* delete is the *soft*) | No |

**The method's *auth* model is the same for all five** (module 36's): the **API key** (the header — the `x-api-key` — module 33's §4) → the **org** (the module-11's tenancy) → the **role** (the *partner's* capability — the module-36's `scopes`: the `products:read`, the `orders:write` — the module-11's RBAC, the *API's* version). **The module-34's line: the method is the *semantics*; the key is the *identity*; the scope is the *permission*** — the *three* are *separate* (the module-11's 401-vs-403: the *no key* = the 401; the *key without the scope* = the 403).

## 3. Architecture — Headers, cookies, and the request's full surface

### 3.1 The `request` object (what you can read, precisely)

| Source | How | The capstone's use |
|---|---|---|
| **URL** (path + query) | `request.url` (the `URL` — the `new URL(request.url).searchParams` for the query) | The module-34's §1's `querySchema` (the module-31's Zod on the query — the *module-34's line: the query is client-supplied (module 31's Zod, module 04's truth)*) |
| **Headers** | `request.headers.get('x-api-key')` (the `Headers` Web API) | The *API key* (module 36), the *`Content-Type`* (the module-34's §3.2's body validation), the *`Sec-Fetch-Mode`* (module 31's dual-path — the *action's* floor detection, *not* the handler's — the *handler's* caller is the *machine* (module 33's Q1) — the *no floor detection here*) |
| **Cookies** | The **`cookies()`** from `next/headers` (the *async* — the module-04's verified: `await cookies()` — the *standard `Request` has no cookies*; the `NextRequest`'s `request.cookies` is the convenience) | The *session* (the module-10's) — the *legitimate* use: a *public* API endpoint that *also* serves the *logged-in app* (the module-36's line: the *dual-auth* (the API key OR the session — the module-33's scenario 4's *note*: the partner API serves the *mobile app* (the key) — the *app's* UI uses the *action* (module 29's) — the *no dual-auth for the app's own data* (the module-16's self-API) — the *dual-auth is for the *public* API (the module-36's line: the *public* API's *logged-in* user's *personalized* data (the module-10's session) + the *partner's* key — the *module-36's scenario: the public API's `/me` (the session) + the `/orgs/:orgId` (the key) — the *two* auth models, *one* API) |
| **Body** | `await request.json()` / `await request.text()` / `await request.formData()` (the *module-34's §3.2's validation*) | The *mutation's* input (module 31's Zod — the *module-34's line: the body is client-supplied (module 31's Zod, module 04's truth)*) |
| **The `params`** (the dynamic segment) | The second argument: `{ params }: { params: Promise<{ orgId: string; id: string }> }` (the *async* — the module-04's verified: the `params` is a *Promise* in 16) | The module-34's §4's `orgId`/`id` (the module-04's boundary validation — the *module-34's line: the params are client-supplied (module 04's Zod) — the *module-33's "derive, don't take"*: the *key's org* is the identity; the *URL's org* is the *validation* (the match — module 33's §4's `keyOrgId !== urlOrgId` → the 403)) |

### 3.2 The body's validation (module 31's Zod, the handler's version)

```ts
// FILE: src/app/api/v1/orgs/[orgId]/products/route.ts — [SERVER] — the POST's body:
const createProductBody = z.object({
  name: z.string().min(1).max(120),
  price: z.coerce.number().min(0).max(999999),   // the module-04's z.coerce (the JSON's string → the number — the module-31's line)
  sku: z.string().max(40).optional(),
  description: z.string().max(2000).optional(),
})

export async function POST(request: Request, { params }: { params: Promise<{ orgId: string }> }) {
  const orgIdFromKey = await getApiKeyOrgId(request)
  if (!orgIdFromKey) return NextResponse.json(ApiErrorBody(401, 'unauthorized', 'Missing or invalid API key'), { status: 401 })
  const { orgId: urlOrgId } = await params
  if (orgIdFromKey !== urlOrgId) return NextResponse.json(ApiErrorBody(403, 'forbidden', 'Key does not match organization'), { status: 403 })   // the module-33's §4's tenancy

  // the module-34's line: the Content-Type check (the module-19's "reject early, reject cheap" — the *module-31's line: the *no parse of a non-JSON body*):
  if (!request.headers.get('content-type')?.includes('application/json')) {
    return NextResponse.json(ApiErrorBody(415, 'unsupported-media-type', 'Expected application/json'), { status: 415 })
  }
  let raw: unknown
  try { raw = await request.json() } catch { return NextResponse.json(ApiErrorBody(400, 'bad-json', 'Invalid JSON body'), { status: 400 }) }

  const parsed = createProductBody.safeParse(raw)
  if (!parsed.success) return NextResponse.json(ApiErrorBody(422, 'validation', 'Invalid body', parsed.error.issues), { status: 422 })

  const created = await createProductService(orgIdFromKey, parsed.data)   // the module-17's service (the *orgIdFromKey* — the *module-33's "derive, don't take"*)
  return NextResponse.json({ data: created }, { status: 201 })   // the module-36's contract: the 201 (the create) — the *data envelope*
}
```

**The status codes** (module 36's taxonomy, the handler's table):

| Status | The meaning | The capstone's use |
|---|---|---|
| `200` | OK (the read / the update) | The `GET`'s list, the `PUT`/`PATCH`'s update |
| `201` | Created (the `POST`'s create) | The `POST /products` (the module-34's §3.2) |
| `202` | Accepted (the *async* — the module-33's challenge's background job) | The `POST /descriptions/generate` (the module-33's challenge's kick-off — the *module-36's line: the 202 is the *async* (the module-23-04's background job) — the *no 200 for the async* (the module-36's contract: the 202 + the `Location` header (the module-08's line: the *async's* *result URL*)) |
| `400` | Bad request (the *malformed* — the JSON parse failure) | The module-34's §3.2's `bad-json` |
| `401` | Unauthorized (the *no identity* — the module-11-03's 401) | The *no API key* (module 36's) |
| `403` | Forbidden (the *identity without the permission* — the module-11-03's 403) | The *key without the scope* (module 36's) / the *key's org ≠ the URL's org* (module 33's §4) |
| `404` | Not found (the *resource* — the module-11-03's "don't leak") | The *product not in the org* (module 17's tenancy — the *module-11-03's 404 (the no leak)* (module 33's line)) |
| `409` | Conflict (the *state* — the module-09-04's idempotency) | The *already-cancelled* (module 33's §4) / the *SKU exists* (module 31's 2b) |
| `415` | Unsupported media type (the *Content-Type*) | The module-34's §3.2's non-JSON |
| `422` | Unprocessable (the *validation* — the module-31's Zod) | The *field errors* (module 36's contract: the `fieldErrors`) |
| `429` | Too many requests (the *rate limit* — module 19's) | The module-36's rate-limit headers (the `Retry-After`) |
| `500` | Server error (the *unexpected* — module 17's AppError 500) | The *module-36's contract: the 500's body is the *generic* (the module-19-03's "no internals") — the *digest* (module 09-04's) in the *logs* (module 21's) |

### 3.3 Streaming responses (the module-08's, the handler's)

The three streaming shapes (the module-27's streaming, the *HTTP* version):

```ts
// (a) The SSE (the module-33's scenario 7 — the live order updates):
export async function GET(request: Request) {
  const stream = new ReadableStream({
    start(controller) {
      const encoder = new TextEncoder()
      const push = (event: string, data: unknown) => {
        controller.enqueue(encoder.encode(`event: ${event}\ndata: ${JSON.stringify(data)}\n\n`))
      }
      push('hello', { at: new Date().toISOString() })
      // the module-17's service's subscription (the module-23-04's background job's event bus — Phase 23):
      const unsubscribe = subscribeOrderEvents(orgId, (evt) => push('order', evt))
      request.signal.addEventListener('abort', () => { unsubscribe(); controller.close() })   // the module-19's line: the *client disconnect* (the `request.signal`) — the *no leak* (module 21's)
    },
  })
  return new Response(stream, { headers: { 'Content-Type': 'text/event-stream', 'Cache-Control': 'no-cache', Connection: 'keep-alive' } })
}

// (b) The file stream (the module-33's scenario 6 — the CSV export):
export async function GET(request: Request, { params }: { params: Promise<{ id: string }> }) {
  const { id } = await params
  const report = await getReportStream(id)   // the module-17's service (the *streaming* generator — the module-18's line: the *no full in-memory* (module 22-01's serverless memory) — the *stream* (module 18's))
  return new Response(report, { headers: { 'Content-Type': 'text/csv', 'Content-Disposition': `attachment; filename="orders-${id}.csv"` } })
}

// (c) The transform (the module-18's line: the *no full in-memory* — the *on-the-fly* transform):
export async function GET(request: Request) {
  const source = await fetch(upstreamUrl)   // the *module-18's line: the *no full in-memory* — the *stream* (module 18's)
  const transformed = source.body!.pipeThrough(new TextDecoderStream()).pipeThrough(/* the transform */)
  return new Response(transformed)
}
```

**The module-34's streaming line:** the *stream is the HTTP response* (module 8's) — the *action can't stream* (module 33's Q2) — the *module-27's streaming is the *HTML* version (the module-12's wire format) — the *handler's streaming is the *HTTP* version (the `ReadableStream`) — the *two are separate* (module 27's §5's rule 5: the *soft-nav streams the RSC payload* (module 12's) — the *handler streams the HTTP* (module 8's) — the *no conflating*).

## 4. Production Code — The webhook, complete (the module-23's §3.3, the handler's full form)

The module-23's §3.3's Stripe webhook is the *canonical* Route Handler — the *module-34's full form* (the *signature first*, the *2xx-fast*, the *revalidateTag*):

```ts
// FILE: src/app/api/webhooks/stripe/route.ts — [SERVER] — the module-23's §3.3, the full:
import { NextResponse } from 'next/server'
import { revalidateTag } from 'next/cache'
import { verifyStripeSignature } from '@/services/stripe'     // the module-19's line: the signature FIRST (module 23's §3.3)
import { markOrderPaid } from '@/services/orders'             // the module-17's service (the *tenancy-aware* — the module-17's rule 1)
import { TAGS } from '@/lib/tags'
import { z } from 'zod'

const eventSchema = z.object({
  id: z.string(),
  type: z.string(),
  data: z.object({ object: z.object({
    id: z.string(),
    metadata: z.record(z.string()).optional(),
  }) }),
})

export async function POST(request: Request) {
  // 1 — the SIGNATURE (the module-19's line: the signature FIRST — the *no parse before the signature* (module 23's §5: the forged webhook's DoS)):
  const signature = request.headers.get('stripe-signature')
  const body = await request.text()   // the *raw* (the signature is over the *raw* — the *no re-serialization* (module 19's line: the JSON parse + re-stringify ≠ the raw — the *module-19's verified*))
  if (!signature) return NextResponse.json({ error: 'missing signature' }, { status: 400 })
  const event = verifyStripeSignature(body, signature)   // the module-17's service (the *HMAC* (module 19's) — the *throws on invalid* (module 19's))
  if (!event) return NextResponse.json({ error: 'invalid signature' }, { status: 400 })

  // 2 — the VALIDATION (the module-31's Zod — the *module-34's line: the body is client-supplied (module 31's Zod, module 04's truth)*):
  const parsed = eventSchema.safeParse(event)
  if (!parsed.success) return NextResponse.json({ error: 'malformed event' }, { status: 400 })

  // 3 — the PROCESSING (the module-23's §3.3's 2xx-fast rule — the *status update inline* (the fast) + the *queue* (the slow — module 23-04's background job)):
  if (parsed.data.type === 'payment_intent.succeeded') {
    const orderId = parsed.data.data.object.metadata?.orderId
    if (orderId) {
      await markOrderPaid(orderId)   // the module-17's service (the *status transition* — the module-09-04's guard — the *idempotent* (module 09-04's line: the *re-delivery* (module 19's line: the *Stripe retries* (module 23's §3.3's note) — the *idempotency is the service's* (module 09-04's guard))
      revalidateTag(TAGS.orders, 'max')
      revalidateTag(TAGS.order(orderId), 'max')
      revalidateTag(TAGS.revenue(orderId /* the orgId — the module-17's service's derived */), 'max')
      // the *slow* (the email, the report) → the *queue* (module 23-04's background job) — the *no inline* (module 23's §3.3's 2xx-fast)
    }
  }

  // 4 — the 2xx-fast (module 23's §3.3's rule — the *Stripe retries on non-2xx* (module 19's line: the *no 500 to Stripe* (module 23's §3.3's note))):
  return NextResponse.json({ received: true })
}
```

**The webhook's *idempotency*** (module 19's line: the *Stripe retries* — the *module-09-04's guard*): the *`markOrderPaid`* is the *idempotent* (the module-09-04's status-transition guard: the *already-paid* → the *no-op* (module 09-04's line) — the *module-34's line: the webhook's idempotency is the service's* (module 09-04's guard) — the *no handler-level idempotency* (module 19's line: the *service is the truth* (module 17's)) — the *module-23's §3.3's note: the *Stripe retries* (module 19's) — the *idempotency is the service's*).

## 5. Common Mistakes (the handler failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The self-API** (the module-16's: the client island's `fetch('/api/…')` to drive the app's UI) | The *RSC re-render bypassed* (module 12's wire format) — the *no floor* (module 30's) — the *BFF's duplicated path* (module 16's) | The *module-29's action* (module 33's Q1: the human, in the app) — the *module-16's self-API* (the module-33's mistake #2) — the *module-34's line: the app's UI uses actions* (module 33's Q1) |
| **The DB in the route** (the Drizzle query *directly* in the handler) | The *module-17's rule violated* (the route is the transport, the service is the logic) — the *tenancy* (module 11's) is *missing* (the *orgId* is *not derived from the key* (module 33's "derive, don't take")) — the *module-34's line: the route calls the service* (module 17's) — the *no DB in the route* (module 17's rule) | The *module-17's service* (module 33's invariant) — the *module-34's line: the route calls the service* (module 17's) — the *no DB in the route* (module 17's rule) |
| **The GET that mutates** (the module-33's mistake #7: the `GET /orders/:id/cancel`) | The *idempotency violated* (module 09-04's) — the *CDN caches the GET* (module 20's L2) — the *mutation is cached* (module 20's L2's violation) | The *POST* (module 29's action / module 8's handler's POST) — the *GET is the read* (module 8's) — the *module-19's line: the no GET mutation* (module 8's) |
| **The full in-memory** (the module-18's line: the *CSV export* that builds the *full* CSV in memory before streaming) | The *serverless's memory* (module 22-01's) — the *10k rows* → the *OOM* (module 18's) | The *stream* (module 34's §3.3's (b)) — the *module-17's service's* *generator* (module 18's line: the *no full in-memory*) — the *module-22-01's serverless memory* (module 22-01's) |
| **The `request.json()` without the try/catch** (the module-34's §3.2's missing) | The *500 on the malformed JSON* (the *module-36's contract: the 400* (the bad-json) — the *500 is the *unexpected* (module 17's AppError 500) — the *malformed JSON is the *expected* (module 36's 400)*) | The *try/catch* (module 34's §3.2) — the *400 bad-json* (module 36's contract) |
| **The session assumed** (the *handler* that reads the *session* for the *app's* data) | The *module-16's self-API* (the read version) — the *no API key* (module 36's) — the *module-34's line: the handler's caller is the machine* (module 33's Q1) — the *session is the action's* (module 29's) | The *module-36's API key* (module 33's Q1: the machine) — the *module-29's action* (module 33's Q1: the human) — the *module-34's line: the handler's auth is the API key* (module 36's) — the *no session for the app's data* (module 16's self-API) |
| **The `Cache-Control` missing** (the *GET* without the *module-36's policy*) | The *default* (the *no `Cache-Control`* → the *browser's heuristic* (module 20's L1) — the *module-36's line: the *explicit* policy* (module 36's contract: the *per-endpoint* `Cache-Control`) — the *module-34's line: the GET's `Cache-Control` is the *module-36's policy* (module 36's) — the *no missing* (module 20's L1's heuristic)) | The *module-36's per-endpoint* `Cache-Control` (module 36's contract) — the *module-34's line: the GET's `Cache-Control` is the module-36's policy* (module 36's) — the *no missing* (module 20's L1's heuristic) |
| **The `request.signal` ignored** (the *SSE* that *doesn't* unsubscribe on the *client disconnect* (module 34's §3.3's (a))) | The *leak* (module 21's) — the *subscription* that *never ends* (module 19's line: the *client disconnect* (module 34's §3.3) — the *no leak* (module 21's)) | The *`request.signal.addEventListener('abort', …)`* (module 34's §3.3's (a)) — the *module-19's line: the client disconnect* (module 34's §3.3) — the *no leak* (module 21's) |

## 6. Security Notes

- **The API key is the header** (module 36's): the *no key in the URL* (module 19's line: the *URL is logged* (module 21's) — the *key in the URL is the *leak* (module 19's)) — the *module-34's line: the key is the `x-api-key` header* (module 36's) — the *module-19's: the key's rotation* (module 19's) — the *module-36's: the key's scopes* (module 11's RBAC).
- **The tenancy is the key's org** (module 33's §4): the *key's org = the URL's org* (module 11-02's cross-tenant) — the *module-34's line: the derive, don't take* (module 33's) — the *key's org is the identity* (module 29's §6's "derive identity from the session" — the API key's version: the derive from the key) — the *module-11's 403 on the mismatch* (module 33's §4).
- **The webhook's signature FIRST** (module 23's §3.3): the *no parse before the signature* (module 23's §5: the forged webhook's DoS) — the *module-34's line: the signature is the first check* (module 23's §3.3) — the *module-19's: the HMAC* (module 19's) — the *module-23's §3.3's verified* (module 23's).
- **The 500's body is generic** (module 36's contract): the *no internals* (module 19-03's) — the *digest* (module 09-04's) in the logs (module 21's) — the *module-34's line: the 500's body is the generic* (module 36's) — the *module-19-03's: the no internals* (module 19-03's).

## 7. Performance Notes

- **The 2xx-fast** (module 23's §3.3): the *webhook's ACK* is the *fast* (module 23's §3.3's rule) — the *slow* (the email, the report) → the *queue* (module 23-04's) — the *module-34's line: the 2xx-fast* (module 23's §3.3) — the *no 500 to Stripe* (module 23's §3.3's note).
- **The streaming** (module 34's §3.3): the *no full in-memory* (module 18's line) — the *serverless's memory* (module 22-01's) — the *module-34's line: the stream is the no full in-memory* (module 18's) — the *module-22-01's serverless memory* (module 22-01's).
- **The `Cache-Control`** (module 36's policy): the *public GETs* are the *CDN-cacheable* (module 20's L2) — the *keyed GETs* are the *no-store* (module 36's line: the header-vary problem) — the *module-34's line: the GET's `Cache-Control` is the module-36's policy* (module 36's).

## 8. Exercise

**Beginner.** *The health endpoint* (module 33's #9): `GET /api/health` — the *200* + the `{"status":"ok"}` — the *module-34's full form* (the auth: the *no auth* (module 33's #9: the *health is the public* (module 19's line: the *no auth for the health* (module 33's #9)) — the *module-34's line: the health is the public* (module 33's #9) — the *no auth*) — the *module-22's deploy's probe* (module 22's) — *build it*, *curl it* (the *200*), *the artifact: the curl's output*.

**Intermediate.** *The partner API's `GET /api/v1/orgs/:orgId/products`* (module 36's public API's *read* — the module-34's §1's pattern, the *products* version): the *API key* (module 36's) → the *org* (module 11's tenancy) → the *listProductsService* (module 17's) → the *envelope* (module 36's) + the *pagination* (module 36's cursor) + the *`Cache-Control`* (module 36's: the *public* → the `s-maxage=60, stale-while-revalidate=300` (module 20's L2)) — *build it*, *curl it* (the *key* + the *query*) — the *cross-tenant test* (module 11-02's: the *org A's key* + the *org B's URL* → the *403*) — the *artifact: the curl's outputs (the 200 + the 403)*.

**Production.** *The webhook's full test* (module 23's §3.3's, the *module-34's* version): the *Stripe's test mode* (module 19's line: the *test mode's webhook* (module 23's §3.3) — the *module-34's line: the webhook's test is the Stripe's test mode* (module 23's §3.3)) — the *payment_intent.succeeded* → the *markOrderPaid* → the *revalidateTag* → the *2xx* — the *re-delivery* (module 19's line: the *Stripe retries* — the *idempotency* (module 09-04's guard) — the *module-34's line: the webhook's idempotency is the service's* (module 09-04's guard)) — the *forged signature* (module 23's §5: the *400* — the *no revalidation* (module 23's §5)) — the *artifact: the three curl's outputs (the 200, the 200 re-delivery, the 400 forged)*.

## 9. Architecture Challenge

**Prompt:** The *"partner's bulk product import"* (the *CSV upload* — the *module-16's file upload*, the *partner's* version): the *partner* uploads a *CSV* (the *1000 products*) — the *module-16's* *file upload* (the *module-30's form field* — the *partner's* *version: the *no form* (the module 30's) — the *module-33's Q1: the machine* (the partner's CLI) — the *module-34's line: the partner's upload is the *Route Handler* (module 33's Q1: the machine) — the *module-16's file upload* (the module 16's) — the *no action* (module 33's Q1: the machine)*) — the *CSV is the *file* (the module 16's) — the *module-34's line: the file is the Route Handler's* (module 33's Q2: the file → the Route Handler) — the *no action for the file* (module 33's Q2)*.

The *problems*: (1) the *1000 products* — the *full in-memory* (module 18's line: the *no full in-memory* (module 22-01's serverless) — the *module-34's line: the *stream* (module 34's §3.3) — the *module-17's service's* *generator* (module 18's)) — the *module-34's line: the 1000 products is the *stream* (module 34's §3.3) — the *no full in-memory* (module 18's line)*.

(2) the *partial failure* (the *450th product's* *SKU exists* — the *module-29's bulk's* *result DTO* (module 29's challenge) — the *partner's* *version: the *no RSC payload* (module 33's Q2: the machine's response) — the *module-34's line: the partner's bulk is the *202* (module 36's: the async) — the *result is the *file* (the CSV's *error report*) — the *module-34's line: the partner's bulk is the *202 + the result file* (module 36's async) — the *no RSC payload* (module 33's Q2)*.

(3) the *rate limit* (module 19's — the *1000 products* — the *module-36's line: the bulk's rate limit is the *per-batch* (module 19's) — the *module-34's line: the bulk's rate limit is the per-batch* (module 19's) — the *no per-product rate limit* (module 19's line: the *batch is the unit* (module 19's))*.

**Design**: the *partner's bulk import* (the *module-34's classification* — the *Q1/Q2/Q3*) — the *Route Handler* (the *module-33's Q1: the machine* — the *module-34's line: the partner's upload is the Route Handler* (module 33's Q1)) — the *file* (the module 16's) — the *stream* (module 34's §3.3) — the *202* (module 36's async) — the *result file* (the CSV's error report) — the *rate limit* (module 19's per-batch) — and the *why* (the *1000 products* → the *stream* (module 34's §3.3) — the *partial failure* → the *result file* (module 36's async) — the *rate limit* → the *per-batch* (module 19's) — the *module-34's standing line: the partner's bulk is the *Route Handler* (module 33's Q1: the machine) + the *stream* (module 34's §3.3) + the *202* (module 36's async) + the *result file* (module 36's) — the *no action for the file* (module 33's Q2) — the *no full in-memory* (module 18's line)*.

<details>
<summary>Model answer</summary>
**The classification** (the module-33's procedure):
- **Q1 the caller**: the *partner's CLI* (the machine) → the *Route Handler* (module 33's Q1) — the *no action* (module 33's Q1: the machine) — the *module-34's line: the partner's upload is the Route Handler* (module 33's Q1).
- **Q2 the response**: the *file* (the CSV) → the *Route Handler* (module 33's Q2: the file → the Route Handler) — the *no action for the file* (module 33's Q2) — the *module-34's line: the file is the Route Handler's* (module 33's Q2).
- **Q3 the cache/team/scale**: the *app's cache* (the *import is a mutation* (module 24's) — the *no cache* (module 20's) — the *module-34's line: the import is the no cache* (module 20's) — the *in-app* (module 33's Q3: the same scale) — the *no external* (module 33's Q3: the same team).

**The door**: the *Route Handler* (the module-33's Q1: the machine) — the *file* (module 16's) — the *stream* (module 34's §3.3) — the *202* (module 36's async) — the *result file* (module 36's) — the *module-34's standing line: the partner's bulk is the Route Handler + the stream + the 202 + the result file*.

**The Route Handler** (the *module-34's full form* — the *POST /api/v1/orgs/:orgId/products/import*):

```ts
// FILE: src/app/api/v1/orgs/[orgId]/products/import/route.ts — [SERVER]:
import { NextResponse } from 'next/server'
import { z } from 'zod'
import { getApiKeyOrgId } from '@/lib/api-auth'   // module 36's API key
import { importProductsStream } from '@/services/products'   // module 17's service (the *streaming* generator — module 18's line: the no full in-memory)
import { enqueueImportReport } from '@/services/reports'   // module 23-04's background job (Phase 23) — the *result file* (module 36's async)

export async function POST(request: Request, { params }: { params: Promise<{ orgId: string }> }) {
  const orgIdFromKey = await getApiKeyOrgId(request)
  if (!orgIdFromKey) return NextResponse.json(ApiErrorBody(401, 'unauthorized', 'Missing or invalid API key'), { status: 401 })
  const { orgId: urlOrgId } = await params
  if (orgIdFromKey !== urlOrgId) return NextResponse.json(ApiErrorBody(403, 'forbidden', 'Key does not match organization'), { status: 403 })

  // the module-16's file (the *formData* — the module-30's §6's multipart — the *partner's* version: the *no form* (module 30's) — the *CLI's* *multipart* (module 16's)):
  const formData = await request.formData()
  const file = formData.get('file')
  if (!(file instanceof File)) return NextResponse.json(ApiErrorBody(400, 'missing-file', 'Expected a file field'), { status: 400 })
  if (file.size > 10 * 1024 * 1024) return NextResponse.json(ApiErrorBody(413, 'file-too-large', 'Max 10MB'), { status: 413 })   // module 19's line: the *size limit* (module 19's)

  // the module-19's per-batch rate limit (module 36's line: the bulk's rate limit is the per-batch (module 19's)):
  await assertRateLimit(orgIdFromKey, 'bulk-import', 10, 3600)   // 10/hour per org (module 19's)

  // the module-23-04's background job (Phase 23) — the *no inline* (module 23's §3.3's 2xx-fast — the *import is the slow* (module 18's line: the 1000 products)):
  const jobId = await enqueueImportReport(orgIdFromKey, file)   // module 23-04's job (the *streaming* import — module 18's line: the no full in-memory)

  // the module-36's 202 (the async) + the Location (module 8's line: the async's result URL):
  return NextResponse.json(
    { data: { jobId, status: 'processing' } },
    { status: 202, headers: { Location: `/api/v1/orgs/${orgIdFromKey}/imports/${jobId}` } },
  )
}
```

**The result endpoint** (the *module-36's async's result* — the *GET /api/v1/orgs/:orgId/imports/:jobId*):

```ts
// FILE: src/app/api/v1/orgs/[orgId]/imports/[jobId]/route.ts — [SERVER]:
export async function GET(request: Request, { params }: { params: Promise<{ orgId: string; jobId: string }> }) {
  const orgIdFromKey = await getApiKeyOrgId(request)
  if (!orgIdFromKey) return NextResponse.json(ApiErrorBody(401, 'unauthorized', 'Missing or invalid API key'), { status: 401 })
  const { orgId, jobId } = await params
  if (orgIdFromKey !== orgId) return NextResponse.json(ApiErrorBody(403, 'forbidden', 'Key does not match organization'), { status: 403 })

  const job = await getImportJob(orgIdFromKey, jobId)   // module 17's service (the *job's status* + the *result file's URL*)
  if (!job) return NextResponse.json(ApiErrorBody(404, 'not-found', 'Import not found'), { status: 404 })

  return NextResponse.json({
    data: {
      status: job.status,   // 'processing' | 'completed' | 'failed'
      imported: job.imported,   // the count
      skipped: job.skipped,     // the count (the module-29's bulk's result DTO — the *partner's* version)
      failed: job.failed,       // the count + the *result file's URL* (module 36's async's result)
      reportUrl: job.status === 'completed' ? job.reportUrl : null,   // the *CSV's error report* (module 36's) — the *module-16's file* (module 16's)
    },
  })
}
```

**The why** (the *three* decisions, the *module's* lines):
1. **The partner's bulk is the Route Handler** (module 33's Q1: the machine) — the *file* (module 16's) — the *stream* (module 34's §3.3) — the *no action for the file* (module 33's Q2) — the *module-34's line: the partner's upload is the Route Handler* (module 33's Q1).
2. **The 1000 products is the stream** (module 34's §3.3) — the *no full in-memory* (module 18's line) — the *module-17's service's generator* (module 18's) — the *module-34's line: the 1000 products is the stream* (module 34's §3.3) — the *module-22-01's serverless memory* (module 22-01's).
3. **The partial failure is the result file** (module 36's async) — the *202* (module 36's) — the *result file* (the CSV's error report — module 36's) — the *module-29's bulk's result DTO* (module 29's challenge) — the *partner's* version: the *no RSC payload* (module 33's Q2: the machine's response) — the *module-34's line: the partner's bulk is the 202 + the result file* (module 36's async).
**The generalization** (the *partner's bulk import's* pattern, the *module's* standing rule): **a *partner's bulk file* (the *1000 products*) is *the Route Handler* (module 33's Q1: the machine) + *the file* (module 16's) + *the stream* (module 34's §3.3: the no full in-memory) + *the 202* (module 36's async) + *the result file* (module 36's: the CSV's error report) + *the per-batch rate limit* (module 19's) — the *no action for the file* (module 33's Q2) — the *no full in-memory* (module 18's line) — the *module-34's standing line: the partner's bulk is the Route Handler + the stream + the 202 + the result file*.
</details>

## 10. Official Documentation

- Route Handlers: https://nextjs.org/docs/app/building-your-application/routing/route-handlers
- Request/Response (the Web APIs — module 01's stack): https://nextjs.org/docs/app/api-reference/functions/next-request
- `NextResponse` (the conveniences): https://nextjs.org/docs/app/api-reference/functions/next-response
- Route Segment Config (`dynamic`, `runtime`, `maxDuration`, `revalidate`): https://nextjs.org/docs/app/building-your-application/configuring/route-segment-config
- `cookies` (the async — module 04's verified): https://nextjs.org/docs/app/api-reference/functions/cookies
- Handling Errors (the 500's digest — module 09-04's): https://nextjs.org/docs/app/guides/handling-errors
- Caching (the `Cache-Control`, the L2 — module 20's): https://nextjs.org/docs/app/getting-started/caching

## 11. What You Should Know Before Continuing

- [ ] I can state the *4 things a Route Handler is not* (module 1's table: the page, the action, the self-API proxy, the DB function) — the *module-33's classifier, enforced*
- [ ] I know the *5-method surface* (module 2's table: the GET/POST/PUT/PATCH/DELETE — the *semantics* + the *cacheability*) — the *module-19's "no GET mutation"* (module 33's mistake #7)
- [ ] I know the *request's full surface* (module 3's: the URL, the headers, the cookies (the `cookies()` — module 04's async), the body, the params (the module-04's Promise)) — the *module-34's line: the query/body/params are client-supplied (module 31's Zod, module 04's truth)*
- [ ] I know the *status codes* (module 3.2's table: the 200/201/202/400/401/403/404/409/415/422/429/500) — the *module-36's contract* (module 36's)
- [ ] I know the *3 streaming shapes* (module 3.3's: the SSE, the file stream, the transform) — the *module-18's line: the no full in-memory* (module 22-01's serverless)
- [ ] I know the *webhook's full form* (module 4's: the signature FIRST, the 2xx-fast, the idempotency (module 09-04's guard)) — the *module-23's §3.3's verified*
- [ ] I've done the *health endpoint* (module 8's beginner) + the *partner API's GET* (module 8's intermediate) + the *webhook's full test* (module 8's production) — the *artifacts: the curl's outputs*
- [ ] I know the *security lines* (module 6's: the API key is the header, the tenancy is the key's org, the signature FIRST, the 500's body is generic)

**Next:** Module 35 — BFF Architecture (the *when to add the layer* — the *module-16's self-API* (the *no BFF for the app's own data*) — the *legitimate BFF* (the module-33's scenario 4: the partner's mobile app) — the *topology* (module 35's) — the *deletion checklist* (module 16's)).
