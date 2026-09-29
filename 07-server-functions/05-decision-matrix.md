# Module 33 — Decision Matrix: Action vs Route Handler vs External API

**Phase 7: Server Functions / Actions · Module 33 of 101 · the final module of Phase 7**

> **Where does this run?** All three options run **`[SERVER]`** — the question is *which door* the client (or a third party) knocks on: the **action's POST** (the module-29's opaque ID, the app's UI), the **Route Handler's URL** (the module-08's `route.ts`, any HTTP client), or the **external API's origin** (a *different* deployment, its own trust boundary). This module is the *classifier* — the 10 scenarios, the decision procedure, and the capstone's surface map.

---

## 1. Concept — Three doors, one service layer

The module-16's "collapsed data path" (component → service → DB) applies to *reads*; for *mutations* and *machine-facing* endpoints, the capstone has **three legitimate doors** into the *same* service layer (module 17):

| Door | The caller | The request | The response | The auth model | The invalidation |
|---|---|---|---|---|---|
| **Server Function (action)** | The **app's UI** (a human, in the browser — the module-29's two invocations: the form, the event handler) | A `POST` to the **action ID** (the opaque reference — the module-12 wire format's up-channel; *only* POST; *sequential* dispatch, module 29) | The **RSC payload** (the single roundtrip: the mutation's result *and* the re-rendered UI — module 29's response model) *or* the 303 (the redirect — the module-30's floor) | The **session** (the module-10's cookie — verified *inside* the action, every time — module 29's "reachable by anyone") | The app's cache (the module-23 tags — `updateTag` for the actor, `revalidateTag` for the fleet) |
| **Route Handler** | **Machines** (webhooks, the partner API, the mobile app, the health checker, the crawler) *and* the app's UI *when the shape is HTTP* (a `GET` download, an SSE stream) | **Any HTTP** (`GET`/`POST`/`…`) to a **URL** (the module-08's `app/api/**/route.ts` — a real, *inspectable* endpoint) | **Whatever the response is** (JSON, a file, a stream, a 2xx ack — the module-08's route handlers) | **Whatever the surface needs** (the session *or* an API key *or* a signature — the module-08-04's per-surface auth; the middleware's first line, module 19) | The app's cache *if in the same app* (the module-23's webhook `revalidateTag` — the module-23's §3.3) — *its own* if external |
| **External API** | **Non-Next consumers at scale**, or a **different team's** domain | HTTP to a **different origin** (a *different* deployment — its own `package.json`, its own DB access, its own caching) | Its own (JSON — the module-08-04's REST/contract) | **A different trust boundary** (API keys, OAuth2 client credentials, mTLS — the module-19's inter-service auth) | **Its own cache** (a *different* tier — the module-20's six layers, *another instance* of them) |

**The invariant (the module's thesis):** *the door is a transport; the service is the logic* (module 17's five contract rules). An action and a Route Handler doing the *same* mutation call the *same* service (the module-17's seam) — the *door* decides the *response shape* (the RSC payload vs the JSON) and the *auth model* (the session vs the key) — *never* the *logic* (the tenancy, the status transition, the audit row — the service's, module 17). **A mutation whose logic lives in the action *and* in the Route Handler (duplicated) is the module-16 self-API anti-pattern, the mutation version** (the module-33's #1 finding).

**The module-16's self-API, the mutation version** (the anti-pattern, precise): an action that *doesn't* call the service — it `fetch()`es *its own app's* Route Handler (`fetch('/api/orders/'+id+'/cancel', {method:'POST'})`) — the *doubled* path (the action → the HTTP → the Route Handler → the service): the *roundtrip* is *doubled* (the module-29's single roundtrip *violated* — the *action's* POST + the *self* HTTP POST), the *logic* is *split* (the action's auth + the handler's auth — *two* gates for *one* truth), the *error* is *doubled* (the HTTP error *between* the action and the handler — the *module-19's* "no error in the middle" — the *action's* `AppError` *becomes* an HTTP status *becomes* the action's *caught* error — the *mapping* is *lost*). **The rule: the action calls the service (the module-29's step 3) — *never* its own Route Handler** (the module-16's deletion checklist, the mutation row: *grep for `fetch('/api/` inside a `'use server'` file — zero*).

## 2. Mental Model — The decision procedure (three questions, in order)

**Q1 — Who calls it?** (the *caller* decides the *door's kind*):
- **A human, in the app's UI** (a form field, a button on a page *the app rendered*) → **an action** (the module-29's — the *form-shaped* → the `<form action>` (module 30); the *client-state-shaped* → the event-handler invocation (module 29's #2)).
- **A machine** (a webhook from Stripe, the partner's mobile app, the health checker, the crawler, the CLI) → **a Route Handler** (the module-08's — a *real URL*, *inspectable*, *any HTTP method*).
- **A non-Next consumer at scale, or a different team's domain** (the public API the *competitors* integrate with; the *billing* service the *other team* owns) → **an external API** (the *different deployment* — the module-22-01's deployment physics: the *scale* is *different* (the public API's traffic ≠ the app's traffic) — the *team* is *different* (the module-23-04's architecture review's "the team owns the service" line)).
- *(The tie-breaker: the **app's UI calling its own machine endpoint** (a `fetch('/api/…')` from a client island) — the **module-16 self-API** (the *read* version) — the *mutation* version is the module-1's anti-pattern — **the app's UI uses actions** (the module-29's) — the *only* legitimate app-UI→Route-Handler is the *shape* the action *can't* express (an `SSE` stream, a `GET` download, a *file* upload's *presigned* URL — the module-33's scenarios 6–8).)*

**Q2 — What comes back?** (the *response shape* confirms Q1's door, or *overrides* it):
- **The re-rendered UI** (the mutation's result *in the page* — the RSC payload) → **an action** (the module-29's single roundtrip — a Route Handler *can't* return the RSC re-render — the *response is HTML/JSON* — the *UI update* is the *client's* manual work (the module-12's wire format *bypassed*) — the *module-33's line: the RSC re-render is the action's *exclusive*)).
- **JSON / a file / a stream / a 2xx ack** (the *machine's* response) → **a Route Handler** (the module-08's).
- *(The override: a *human* in the UI, but the *response is a file* (the "download the CSV" — the *module-33's scenario 6*) → the *Route Handler* (the *action* can't stream a file to the browser's download manager — the *response is the RSC payload* (the module-29's) — the *file is the HTTP response* (the module-08's) — the *module-33's line: the response shape overrides the caller* (the *file/stream* → the Route Handler) — the *action kicks off the work; the Route Handler delivers the artifact* (the module-33's scenario 6's pattern) — the *no action-for-files*).

**Q3 — Whose cache, whose team, whose scale?** (the *deployment physics* — the module-20's six layers + the module-22-01's deployment):
- **The app's cache** (the module-23 tags — the `revalidateTag` the *webhook* calls — the module-23's §3.3) → **in the same app** (a Route Handler *in* `app/api/**` — the *same* cache tier — the *module-33's line: the webhook's `revalidateTag` works *because* the webhook is *in the app* (the same tier) — an *external* API's invalidation is *its own* (a *different* tier — the module-20's L3, *another instance*)).
- **A different team's domain** (the *billing* — the *module-23-04's review's line*) → **an external API** (the *different deployment* — the *module-22-01's* — the *team owns the service* (the module-17's "the service is the seam" — the *external* API is *another team's seam*).
- **A different scale** (the *public* API's 10k rps ≠ the app's 100 rps — the module-18's performance) → **an external API** (the *different deployment's* scaling — the *module-22-01's* — the *public* API's *cache* is *its own* (a CDN + a *different* L3 — the module-20's layers, *another instance*).

**The one-line procedure** (the module's standing rule): **Q1 the caller (human→action; machine→handler; scale/team→external) — Q2 the response (RSC→action; file/stream/JSON→handler) — Q3 the cache/team/scale (app's→in-app; different→external) — the *service is the logic* (module 17) — the *door is the transport*.**

## 3. Architecture — The capstone's surface map (the 10 scenarios, classified)

`FILE: docs/surface-map.md` (the artifact — the capstone's *every* mutation + machine endpoint, classified — the module-33's matrix, *filled*):

| # | Scenario (the capstone's) | Q1 caller | Q2 response | Q3 cache/team/scale | **The door** | The rationale (the one-line why) |
|---|---|---|---|---|---|---|
| 1 | *Cancel an order* (the dashboard's `OrdersTable` — the module-25's) | A human, in the app (a button) | The re-rendered UI (the *status* changes — the RSC payload) | The app's cache (the `'orders'` tag — module 23) | **Action** (the event-handler — module 29's #2) | The RSC re-render is the action's exclusive (module-1's invariant); the module-23's `updateTag` (the actor's immediate) |
| 2 | *Stripe's payment webhook* (the `payment_intent.succeeded` — module 23's §3.3) | A machine (Stripe) | A 2xx ack (the *fast* — module 23's "2xx-fast rule") | The app's cache (the `revalidateTag` — module 23's §3.3) | **Route Handler** (`POST /api/webhooks/stripe`) | The machine's POST; the 2xx ack; the *signature* auth (module 23's `verifyStripeSignature`); the app's cache (the `revalidateTag` — in-app) |
| 3 | *The "notify me" contact form* (the (marketing) public page — module 24's SHAPE 1) | A human, *public* (a form) | The 303 → the "thanks" page (the *floor* — module 30's) | The app's cache (the *no tag* — the *contact is a *write* (no read to invalidate)) | **Action** (the form — module 30's) | The *form-shaped* human → the action (module-2's Q1); the *floor* (the JS-off — module 30's); the *public* (the *no session* — the *rate limit* (module 19's) — the action's *check* (the module-19's line: the public action's rate limit — the *module-33's security note*)) |
| 4 | *The partner REST API* (the *mobile app's* order list — the module-16's "third party" — the module-29's challenge's "partner API") | A machine (the *mobile app*) | JSON (the *REST* — module 08-04's) | The app's cache *or* the *external* (the *scale* — the *module-33's Q3: the mobile's traffic = the app's traffic? — the *capstone's* answer: the *in-app* Route Handler (the *same* scale — the *module-22-01's* — the *no external* (the *over-engineering* — the module-33's common mistake #5)) | **Route Handler** (`GET /api/v1/orgs/:orgId/orders` — the module-08-04's) | The machine's GET; the JSON; the *API key* auth (the module-08-04's); the *in-app* (the *same* scale — the *no external*) — the *the* *the the *service is the same* (the module-17's `getOrderListCached` (the module-25's) — the *door is the JSON* (the module-08's) — the *no self-API* (the handler calls the *service* (module 17) — *not* the action) |
| 5 | *Upload a product image* (the product form's `image` field — module 16's) | A human, in the app (a `<input type="file">`) | The re-rendered UI (the *image's* URL — the RSC payload) | The app's cache (the `'products'` tag — module 23) | **Action** (the form field — module 30's §6: the *multipart* — the module-16's) | The *form-shaped* (the `<input type="file">` is a *form field* (module 30's) — the *floor* (the JS-off's *native* multipart POST — module 16's); the *service* (the module-16's upload — the *S3/Cloudinary* (the module-16's) — the *action's step 3* (module 29's) — the *image's URL is the DTO* (module 17's) |
| 6 | *Download the orders CSV* (the "export" button — the *long-running* report — the module-18's) | A human, in the app (a button) — *but the response is a file* | **A file** (the CSV — the *browser's download manager*) | The app's cache (the *report is a *compute* (no read to invalidate) — the *module-18's* — the *report's* *generation is the *Route Handler's* (the *module-33's Q2: the file → the Route Handler*)) | **Action (kick off) + Route Handler (deliver)** (the *two-step* — the module-33's scenario 6's pattern: the *action* *enqueues* the job (the module-23-04's background job — Phase 23) + the *redirects* to the *download URL*; the *Route Handler* *serves* the file (the `GET /api/reports/:id.csv` — the module-08's) — the *no action-for-files* (module-2's Q2: the response shape overrides the caller)) | The *file is the HTTP response* (the module-08's) — the *action can't stream a file* (module-2's Q2) — the *two-step* (the action kicks off; the handler delivers) — the *module-33's line: the action-for-files is the anti-pattern* (the *module-2's Q2: the response shape overrides*) |
| 7 | *Live order updates* (the "orders changed" SSE stream — the module-28's challenge's "SSE upgrade") | A human, in the app (the *browser's* `EventSource`) — *but the response is a stream* | **A stream** (the SSE — the `text/event-stream` — the module-08's route handler's *streaming response*) | The app's cache (the *stream is a *live* (no cache — the module-20's NOT-CACHED)) | **Route Handler** (`GET /api/orders/stream` — the module-08's *streaming*) | The *stream is the HTTP response* (the module-08's) — the *action can't stream* (module-2's Q2) — the *SSE is the Route Handler's* (the module-08's) — the *module-33's line: the stream → the Route Handler* (the *module-2's Q2: the response shape overrides*) |
| 8 | *Generate a product thumbnail* (the on-demand image resize — the module-15's `next/image`'s *alternative* — the *module-15's* `next/image` is the *default* (the module-15's) — the *custom* resize is the Route Handler's) | A machine (the *`next/image`'s* *fallback* — the *browser's* `<img>`) — *but the response is an image* | **An image** (the resized — the *module-15's* — the *Route Handler's* *image response*) | The app's cache (the *thumbnail is a *compute* (the *module-15's* — the *cache is the *CDN* (the module-20's L2) — the *Route Handler's* *response is cacheable* (the module-08's) — the *CDN's* *cache* (the module-20's L2)) | **Route Handler** (`GET /api/thumbnails/:productId` — the module-08's *image response* — the *module-15's* — the *`next/image`'* *`loader`* (the module-15's) — the *no action* (the *machine's* `<img>` (Q1) + the *image response* (Q2))) | The *`<img>` is a machine* (Q1) — the *image is the response* (Q2) — the *Route Handler's* (the module-08's) — the *module-15's* `next/image`'s *`loader`* (the module-15's) — the *no action* (the *no human* (Q1)) |
| 9 | *The health check* (the `/api/health` — the *deploy's* probe — the module-22's) | A machine (the *load balancer's* probe) | A 200 (the *`{status:'ok'}`* — the module-08's) | The *no cache* (the *health is a *live* (the module-20's NOT-CACHED) — the *module-22's* — the *the* *the the *health is the *Route Handler's* (the module-08's) — the *no action* (the *no human* (Q1))) | **Route Handler** (`GET /api/health` — the module-08's) | The *machine's* GET (Q1) — the *200 is the response* (Q2) — the *Route Handler's* (the module-08's) — the *module-22's* deploy's probe — the *no action* (the *no human*) |
| 10 | *The "compare with last period" toggle* (the module-25's challenge's *view arg* — the *module-25's* *view args join the key*) | A human, in the app (a toggle) — *but the *response is the *re-render* (the *analytics section* changes) — the *module-25's challenge's answer: the *view arg is the *URL state* (the module-10's) — the *server renders the compare view* (the module-24's pattern) — the *no mutation* (the *toggle is a *read* (the module-24's decision table) — the *no action* (the *no mutation*) — the *no Route Handler* (the *no machine*) — the *the* *the the *toggle is the *URL state* (the module-10's) — the *page's re-render* (the module-24's) — the *module-33's line: the *view arg is the *URL state* (the module-10's) — the *no door* (the *no mutation* — the *module-33's scenario 10: the *not a mutation* (the *no action/handler/external* — the *module-33's matrix's "N/A" row*)** | *N/A* (the *re-render* — the *module-24's pattern* — the *no door* (the *no mutation*)) | The app's cache (the *`'analytics:{orgId}'` tag — module 25's — the *view arg joins the key* (module 25's)) | **N/A** (the *no door* — the *URL state* (the module-10's) — the *page's re-render* (the module-24's) — the *module-33's line: the *view arg is the *URL state* (the module-10's) — the *no mutation* — the *no door*) | The *toggle is a *read* (the module-24's decision table) — the *no mutation* (the *no action*) — the *URL state* (the module-10's) — the *page's re-render* (the module-24's) — the *module-25's challenge's answer: the *view arg joins the key* (the module-25's) — the *no door* (the *module-33's N/A row*) |

**The map's rule (the module's gate):** *every* mutation + machine endpoint in the capstone is *classified* (the *10 scenarios* — the *module-33's matrix*) — the *no "we'll decide later"* (the *module-20's inventory's rule: the *no missing row*) — the *N/A row* (scenario 10) is *legitimate* (the *no mutation* — the *URL state*) — the *module-33's standing line: the *surface map is the *artifact* (the *module-20's inventory's* *mutation* version) — the *module-23-04's architecture review's row* (the *surface map is the review's artifact*)*.

## 4. Production Code — The one mutation, three doors (the *order cancel*, the *contrast*)

**Door 1 — the action** (the module-29's `cancelOrder` — the *module-4's* complete — the *app's UI*'s door): the module-29's §4's `cancelOrder` (the five steps — the session → validate → service → invalidate → redirect) — the *RSC payload* (the module-29's single roundtrip) — the *module-33's line: the action is the *app's UI's* door (the module-2's Q1: the human, in the app) — the *RSC re-render is the action's exclusive* (module-2's Q2: the RSC → the action)*.

**Door 2 — the Route Handler** (the *partner API's* `POST /api/v1/orgs/:orgId/orders/:orderId/cancel` — the module-08-04's — the *machine's* door):

`FILE: src/app/api/v1/orgs/[orgId]/orders/[orderId]/cancel/route.ts` (production pattern — [SERVER] — the module-08-04's)

```ts
import { NextResponse } from 'next/server'
import { z } from 'zod'
import { cancelOrderService } from '@/services/orders'   // the module-17's service (the *same* service as the action (module 1's invariant) — the *no duplication*)
import { TAGS } from '@/lib/tags'
import { revalidateTag } from 'next/cache'

// the module-08-04's API key auth (the *machine's* auth — the *no session* (Q1: the machine) — the module-19's line: the *API key is the *partner's* (the module-08-04's)):
async function getApiKeyOrgId(request: Request): Promise<string | null> {
  const apiKey = request.headers.get('x-api-key')
  if (!apiKey) return null
  // the module-08-04's API key lookup (the *module-17's service* — the `getOrgByApiKey` (the module-17's) — the *module-19's line: the *API key is the *partner's* (the module-08-04's)) — the *no session* (Q1: the machine):
  const orgId = await getOrgByApiKey(apiKey)   // the module-17's service (the *module-08-04's* — the *module-19's* — the *API key's* *org* (the *module-11's tenancy*))
  return orgId
}

export async function POST(
  request: Request,
  { params }: { params: Promise<{ orgId: string; orderId: string }> },
) {
  const { orgId: urlOrgId, orderId } = await params
  // the module-19's line: the *API key's org* must match the *URL's org* (the *module-11's tenancy* — the *no cross-tenant* (module 11-02's)):
  const keyOrgId = await getApiKeyOrgId(request)
  if (!keyOrgId) return NextResponse.json({ error: 'Unauthorized' }, { status: 401 })   // the module-11-03's 401 (the *no key*)
  if (keyOrgId !== urlOrgId) return NextResponse.json({ error: 'Forbidden' }, { status: 403 })   // the module-11-03's 403 (the *wrong org*) — the *module-19's line: the *no cross-tenant* (module 11-02's))

  const id = z.string().uuid().safeParse(orderId)
  if (!id.success) return NextResponse.json({ error: 'Invalid order ID' }, { status: 422 })

  // the module-17's service (the *same* service as the action (module 1's invariant) — the *no duplication*) — the *module-29's step 3* (the service is the logic):
  try {
    await cancelOrderService(keyOrgId, id.data)   // the *keyOrgId* (the *API key's org* — the *module-11's tenancy* — the *no URL's org* (the *module-19's line: the *derive, don't take* — the *key's org is the *identity* (the module-29's §6's "derive identity from the session" — the *API key's* version: the *derive from the key*))
  } catch (e) {
    if (isAppError(e) && e.code === 'already-cancelled') {
      return NextResponse.json({ error: 'Already cancelled' }, { status: 409 })   // the module-11-03's 409 (the *module-31's expected* — the *handler's* *version* (the *module-08's JSON*)
    }
    if (isAppError(e) && e.code === 'not-found') {
      return NextResponse.json({ error: 'Not found' }, { status: 404 })   // the module-11-03's 404 (the *no leak*)
    }
    throw e   // the *unexpected* (the 500) — the *module-08's* error (the *module-19's* — the *no internals*)
  }

  // the module-23's invalidation (the *fleet's* SWR — the *no `updateTag`* (the module-23's: the *`updateTag` is the action's* (the module-23's §1: the *actor's* immediate) — the *handler has no actor* (Q1: the machine) — the *`revalidateTag` only* (module 23's §3.3: the webhook's pattern)):
  revalidateTag(TAGS.orders, 'max')
  revalidateTag(TAGS.order(id.data), 'max')
  revalidateTag(TAGS.revenue(keyOrgId), 'max')

  return NextResponse.json({ ok: true })   // the module-08's JSON (Q2: the machine's response) — the *no RSC payload* (the action's exclusive — module-2's Q2)
}
```

**Door 3 — the external API** (the *public* API the *competitors* integrate with — the *different deployment* — the *module-22-01's*): the *no code in this app* (the *different deployment's* code — the *module-22-01's* — the *module-23-04's review's line: the team owns the service*) — the *module-33's line: the external API is the *different deployment* (the module-22-01's) — the *no code in this app* (the *module-23-04's review's line: the team owns the service*) — the *module-33's scenario 4's tie-breaker: the *in-app* Route Handler (the *same* scale) — the *external* is the *different scale/team* (Q3) — the *module-33's common mistake #5: the *external for one internal feature* (the *over-engineering*)*.

**The contrast** (the *module-33's table*, the *one mutation*):

| | **Action** (Door 1) | **Route Handler** (Door 2) | **External API** (Door 3) |
|---|---|---|---|
| The caller | The app's UI (the human) | The machine (the partner) | The non-Next consumer at scale (the competitor) |
| The request | The POST to the action ID (the module-29's) | The POST to the URL (the module-08's) | The POST to the *different origin* (the module-22-01's) |
| The response | The RSC payload (the module-29's single roundtrip) | The JSON (the module-08's) | The JSON (the *different deployment's*) |
| The auth | The session (the module-10's) | The API key (the module-08-04's) | The *different trust boundary* (the module-19's) |
| The invalidation | The `updateTag` (the actor) + the `revalidateTag` (the fleet) (module 23's) | The `revalidateTag` (the *no `updateTag`* — the *no actor*) (module 23's §3.3) | The *different deployment's* cache (the module-20's L3, *another instance*) |
| The service | The `cancelOrderService` (module 17's) | The *same* `cancelOrderService` (module 1's invariant) | The *different deployment's* service (the *module-23-04's review's line*) |
| The error | The `AppError` → the module-31's state / the module-27's boundary | The `AppError` → the module-08's JSON status | The *different deployment's* error (the module-19's) |

**The module-33's standing line:** *the door is the transport; the service is the logic* (module 17's) — the *action and the Route Handler call the same service* (module 1's invariant) — the *no duplication* (the module-16's self-API, the mutation version — module 1's anti-pattern) — the *module-23-04's architecture review's row: the *surface map is the review's artifact* (the *10 scenarios* — the *module-33's matrix*)*.

## 5. Common Mistakes (the decision failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The self-API mutation** (the module-1's anti-pattern: the action `fetch()`es its own Route Handler) | The *doubled* roundtrip (the module-29's single roundtrip violated) — the *split* logic (two gates) — the *doubled* error (the HTTP error in the middle — the module-19's "no error in the middle") | The *module-29's step 3: the action calls the service* (module 17's) — the *no self-API* (module 16's deletion checklist, the mutation row: the *grep for `fetch('/api/` inside a `'use server'` file — zero*) |
| **The Route Handler for the app's UI** (a `fetch('/api/orders/cancel')` from a client island — the module-16's self-API, the read version, the mutation's analog) | The *RSC re-render bypassed* (the module-12's wire format — the *manual* client state update — the module-12's "the island contract" violated) — the *no floor* (the JS-off — module 30's) — the *BFF's duplicated path* (module 16's) | The *module-29's action* (the app's UI's door — module-2's Q1: the human, in the app) — the *RSC re-render is the action's exclusive* (module-2's Q2) — the *module-16's self-API* (the read version) — the *module-33's line: the app's UI uses actions* (module-2's Q1) |
| **The action for a file** (the "download the CSV" as an action — module-33's scenario 6) | The *action can't stream a file* (module-2's Q2: the response is the RSC payload — the *file is the HTTP response* (module 08's)) — the *the* *the the *action's response is the RSC payload* (module 29's) — the *no file* (module-2's Q2) | The *module-33's scenario 6's pattern: the action kicks off the job (the module-23-04's background job) + the redirect to the download URL; the Route Handler serves the file* (the module-08's) — the *module-33's line: the response shape overrides the caller* (module-2's Q2: the file/stream → the Route Handler) — the *no action-for-files* |
| **The action for a stream** (the SSE as an action — module-33's scenario 7) | The *action can't stream* (module-2's Q2: the response is the RSC payload — the *stream is the HTTP response* (module 08's)) | The *Route Handler's streaming response* (the module-08's `text/event-stream`) — the *module-33's line: the stream → the Route Handler* (module-2's Q2) |
| **The Route Handler that returns HTML** (a `GET /api/page` that returns a *page* — the module-08's route handler's *response is JSON/file/stream* (module 08's) — the *HTML is the route's* (module 05's)) | The *confusion* (the *API* that returns a *page* — the *module-08's line: the Route Handler is the *machine's* (Q1) — the *HTML is the human's* (Q1: the page) — the *the* *the the *Route Handler that returns HTML is the *page* (module 05's) — the *module-33's line: the HTML → the route (module 05's) — the *no Route Handler for HTML*) | The *module-05's route* (the `app/**/page.tsx` — the *human's* door) — the *Route Handler is the machine's* (Q1) — the *no HTML in a Route Handler* (module 08's) |
| **The external API for one internal feature** (the module-33's scenario 4's tie-breaker violated: the *mobile app's* API as an *external* deployment — the *over-engineering* (module-33's common mistake #5)) | The *different deployment's* cost (the module-22-01's — the *different team's* ops — the module-23-04's review's "the team owns the service" line — the *no team for the external API* (the *one team* (the capstone's) — the *module-33's line: the external is the *different team* (Q3) — the *no team = the *no external*) | The *module-33's scenario 4's tie-breaker: the in-app Route Handler* (the *same* scale — the module-22-01's) — the *no external* (the over-engineering) — the *module-33's line: the external is the *different scale/team* (Q3) — the *one team = the in-app* |
| **The GET Route Handler that mutates** (a `GET /api/orders/:id/cancel` — the *module-19's line: the GET is the *idempotent read* (module 08's) — the *mutation is the POST* (module 29's) — the *the* *the the *GET that mutates is the *module-19's* "no GET mutation" (module 08's) — the *module-33's line: the mutation → the POST* (module 29's) — the *no GET mutation*) | The *idempotency violated* (the module-09-04's — the *GET's* *cache* (the module-20's L2 — the CDN) — the *the* *the the *CDN caches the GET* (the module-20's L2) — the *the* *the the *the mutation is *cached* (the *module-20's L2's* *violation*) — the *module-33's line: the mutation → the POST* (module 29's) — the *no GET mutation*) | The *POST* (module 29's action / module 08's Route Handler's POST) — the *GET is the read* (module 08's) — the *module-19's line: the no GET mutation* (module 08's) |
| **The "we'll decide later"** (the *surface map's missing row* — module-33's map's rule violated) | The *undecided* mutation (the *module-20's inventory's rule: the no missing row*) — the *the* *the the *mutation is *built* (the *module-29's action) — the *the* *the the *decision is *the action* (the *default*) — the *module-33's line: the surface map is the artifact* (module 20's inventory's mutation version) — the *no missing row*) | The *module-33's 10 scenarios* (the *classified*) — the *surface map* (the *docs/surface-map.md* — the *module-23-04's review's artifact*) — the *no missing row* (module 20's rule) |

## 6. Security Notes

- **The action's public route** (module 29's §6): the *action ID is in the client bundle* (visible) — the *auth is the action's* (module 29's step 1) — the *module-33's line: the action's security is the action's* (module 29's §6) — the *no middleware shortcut* (module 10-05's) — the *module-19's threat model: the *scanner finds the action IDs* (module 29's §6) — the *module-33's scenario 3's public action* (the "notify me" — the *no session* — the *rate limit* (module 19's) — the *action's check* (module 19's line: the public action's rate limit) — the *module-33's security note: the *public action's rate limit is the *action's* (module 19's) — the *no form for the public without a rate limit* (module 19's))*.
- **The Route Handler's API key** (module 08-04's): the *API key is the partner's* (module 08-04's) — the *module-19's line: the API key is the *secret* (module 19's) — the *no API key in the URL* (module 19's) — the *module-33's line: the API key is the header* (the `x-api-key` — module 08-04's) — the *module-19's: the API key's rotation* (module 19's) — the *module-33's scenario 4's* `getApiKeyOrgId` (module 08-04's) — the *module-11's tenancy: the key's org = the URL's org* (module 11-02's cross-tenant — the module-4's `keyOrgId !== urlOrgId` → the 403)*.
- **The external API's trust boundary** (module 19's): the *different deployment's* auth (module 19's) — the *module-23-04's review's line: the team owns the service* (module 23-04's) — the *module-33's line: the external API's trust boundary is the *different deployment's* (module 19's) — the *module-22-01's deployment physics: the *different scale/team* (Q3) — the *module-33's scenario 4's tie-breaker: the in-app (the same scale) — the external (the different scale/team* (Q3)*.
- **The webhook's signature** (module 23's §3.3): the *Stripe's signature* (module 23's `verifyStripeSignature`) — the *module-19's line: the signature is the *webhook's auth* (module 19's) — the *module-23's §3.3's "signature check FIRST"* (module 23's) — the *module-33's scenario 2's* `POST /api/webhooks/stripe` (module 23's §3.3) — the *module-19's: the *forged webhook* (module 23's §5: the "a forged webhook that revalidates tags is a cheap DoS" (module 23's §5) — the *module-33's line: the webhook's signature is the *first* check* (module 23's §3.3) — the *no revalidation before the signature* (module 23's §5)*.

## 7. Performance Notes

- **The action's single roundtrip** (module 29's): the *mutation + the re-render* in *one* POST (module 29's) — the *module-18-01's measurement: the action's TTFB* (module 29's §7: the 20–100ms) — the *module-33's line: the action's performance is the single roundtrip* (module 29's) — the *no self-API* (module 1's anti-pattern: the *doubled* roundtrip — module 29's §7's "the roundtrip count is halved" line, the *self-API's* violation).
- **The Route Handler's response** (module 08's): the *JSON/file/stream* (module 08's) — the *module-18-01's measurement: the handler's TTFB* (module 08's) — the *module-33's line: the handler's performance is the response's* (module 08's) — the *module-20's L2: the CDN's cache* (module 20's) — the *module-33's scenario 8's thumbnail* (module 15's) — the *CDN's cache* (module 20's L2) — the *module-33's line: the handler's response is cacheable* (module 08's) — the *CDN's cache* (module 20's L2)*.
- **The external API's scale** (module 22-01's): the *different deployment's* scaling (module 22-01's) — the *module-18's performance: the public API's 10k rps* (module 18's) — the *module-33's line: the external API's performance is the *different deployment's* (module 22-01's) — the *module-33's scenario 4's tie-breaker: the in-app (the same scale) — the external (the different scale) (Q3)*.

## 8. Exercise

**Beginner.** *Classify the 10 scenarios* (the module-33's matrix) — *from memory* (the *no peeking*) — the *Q1/Q2/Q3* for *each* — the *door* — the *one-line why* — *then check* (the *module-33's §3's table*) — the *the ones you got wrong* (the *the* *the the *the procedure's step you missed* (Q1/Q2/Q3) — the *module-33's line: the *procedure is the artifact* (module 2's) — the *no "I felt it"* (module 20's inventory's rule: the *reasoning is the artifact*) — *screenshot your classifications* (the *the* *the the *the module-20's testing phase's starting point* (module 20's) — the *module-33's matrix is the test's starting point*).

**Intermediate.** *The surface map* (the *docs/surface-map.md*) — *every* mutation + machine endpoint in the capstone (the *module-29's action inventory* + the *module-08's route handlers* + the *module-23-04's background jobs* (Phase 23) — the *10 scenarios* + the *capstone's actual surfaces*) — the *Q1/Q2/Q3* per row — the *door* — the *rationale* — the *the* *the the *the missing rows* (the *module-20's inventory's rule: the no missing row*) — the *module-23-04's architecture review's artifact* (module 23-04's) — the *screenshot the map*.

**Production.** *The decision audit* (the *module-19's "every endpoint" audit*, the *decision* version): for *every* door in the capstone (the *actions* + the *Route Handlers* + the *external APIs*) — *verify*: (a) the *Q1* (the caller matches the door — the *no action for a machine* — the *no handler for the app's UI* (module 16's self-API) — the *module-33's common mistake #2*) (b) the *Q2* (the response matches the door — the *no action for a file/stream* (module-33's common mistake #3/#4) — the *no handler for HTML* (module-33's common mistake #5)) (c) the *Q3* (the cache/team/scale matches — the *no external for one internal feature* (module-33's common mistake #6) — the *no in-app for a different team's domain* (Q3)) (d) the *service is the logic* (module 1's invariant: the *action and the handler call the same service* — the *no duplication* (module 16's self-API)) — the *four checks per door* (the *module-19's audit's row* — the *module-20's testing phase's starting point*) — the *artifact: the docs/surface-audit.md* (the *per-door* results — the *module-23-04's review's artifact*).

## 9. Architecture Challenge

**Prompt:** The *"AI product description generator"* (the *new feature*): the *product form* (the module-30's) gets a *"Generate description"* button (the *client island* — the module-13's) — the *click* → the *AI's API* (the *module-18's external* — the *the* *the the *OpenAI's/Anthropic's* — the *module-18's external's cost* (module 18's) — the *the* *the the *generation is the *slow* (the *5–30s* — the module-18's external) — the *the* *the the *response is the *text* (the *description* — the *module-14's DTO* — the *the* *the the *form's field is *filled* (the *module-31's useActionState's state* — the *description field*) — the *the* *the the *the user's edit is the *form's* (the module-31's preserved values) — the *the* *the the *the "Generate" is the *no mutation* (the *no DB write* — the *the* *the the *generation is a *compute* (the module-24's decision table: the *no change rate* — the *no tag* — the *no invalidation*) — the *module-33's line: the "Generate" is the *no door* (the *no mutation* — the *module-33's scenario 10's N/A row*) — the *the* *the the *the "Generate" is the *action* (the *module-29's event-handler* (module 29's #2) — the *the* *the the *response is the *RSC payload* (the module-29's) — the *the* *the the *the form's field is *filled* (the module-31's state) — the *module-33's line: the "Generate" is the *action* (module 29's #2) — the *response is the RSC payload* (module 29's) — the *form's field is filled* (module 31's state*)*.

The *problems*: (1) the *generation is the slow* (the *5–30s* — the module-18's external) — the *action's timeout* (the module-18-01's — the *the* *the the *serverless's timeout* (module 22-01's) — the *module-18's line: the external's cost* (module 18's) — the *the* *the the *5–30s is the *serverless's timeout* (module 22-01's) — the *module-33's line: the slow generate is the *background job* (module 23-04's) — the *no action for the slow* (module-18's line: the external's cost) — the *module-33's scenario 6's pattern: the action kicks off the job + the Route Handler delivers* (module 33's §3's scenario 6) — the *module-33's line: the slow compute → the background job* (module 23-04's) — the *no action for the slow*).

(2) the *AI's API cost* (the module-18's external's cost) — the *rate limit* (module 19's) — the *per-user* (module 11's tenancy) — the *module-18's line: the external's cost* (module 18's) — the *module-33's line: the AI's cost is the *rate limit* (module 19's) — the *per-user* (module 11's) — the *module-33's security note: the *AI's rate limit is the *action's* (module 19's) — the *no generate without a rate limit* (module 19's)*.

(3) the *the user's edit* (the module-31's preserved values) — the *the* *the the *the generated text is the *form's field* (the module-31's) — the *the* *the the *the user's edit is the *form's* (module 31's) — the *module-33's line: the generated text is the *form's field* (module 31's) — the *the user's edit is the form's* (module 31's) — the *no separate state* (module 13's line: the form owns the state)*.

**Design**: the *"Generate"* (the *module-33's scenario 10's N/A row* — the *no mutation* — the *the* *the the *the "Generate" is the *action* (module 29's #2) — the *response is the RSC payload* (module 29's) — the *form's field is filled* (module 31's state) — the *module-33's line: the "Generate" is the action* (module 29's #2) — the *slow* (module 18's external) — the *module-33's scenario 6's pattern: the action kicks off the job (module 23-04's) + the Route Handler delivers (module 08's)* — the *no action for the slow* (module 18's line) — the *module-33's standing line: the slow compute → the background job* (module 23-04's) — the *no action for the slow*).

Produce: the *"Generate"* (the *module-33's classification* — the *Q1/Q2/Q3*) — the *action* (the module-29's five steps — the *non-redirecting* (module 29's §4's exception: the return the DTO (the generated text)) — the *slow* (module 18's external) — the *module-33's scenario 6's pattern: the action kicks off the job (module 23-04's) + the redirect to the "generating" page + the SSE stream (module 33's scenario 7) delivers the text* — the *module-33's line: the slow generate → the background job + the SSE* (module 23-04's + module 08's) — the *no action for the slow*) — the *rate limit* (module 19's — the per-user — module 11's tenancy) — the *the user's edit* (module 31's preserved values) — and the *why* (the *slow* (module 18's external) — the *no action for the slow* (module-18's line) — the *the* *the the *the background job* (module 23-04's) — the *SSE* (module 08's) — the *module-33's standing line: the slow compute → the background job + the SSE* (module 23-04's + module 08's) — the *no action for the slow*).

<details>
<summary>Model answer</summary>
**The classification** (the *module-33's procedure*):
- **Q1 the caller**: the *human, in the app* (a button) → the *action* (module 29's #2) — the *but the response is the *slow stream* (the *5–30s* — module 18's external) — the *module-33's Q2: the response shape overrides the caller* (module 2's Q2: the stream → the Route Handler) — the *module-33's line: the slow stream → the Route Handler* (module 08's SSE) — the *no action for the slow* (module 18's line) — the *module-33's scenario 6's pattern: the action kicks off the job (module 23-04's) + the Route Handler delivers (module 08's)*.
- **Q2 the response**: the *slow stream* (the *5–30s* — the *SSE* (module 08's) — the *module-33's Q2: the stream → the Route Handler*) — the *module-33's line: the response shape overrides the caller* (module 2's Q2) — the *no action for the slow* (module 18's line).
- **Q3 the cache/team/scale**: the *app's cache* (the *no tag* — the *generation is a compute* (module 24's decision table: the no change rate — the no tag — the no invalidation) — the *module-33's line: the generate is the no invalidation* (module 20's five questions: the WHO invalidates is no one (module 25's NOT-CACHED row) — the *the* *the the *the generation is the *compute* (module 24's) — the *no cache* (module 20's) — the *module-33's line: the generate is the no cache* (module 20's) — the *the* *the the *the AI's output is the *form's field* (module 31's) — the *no cache*).

**The door**: the *action (kick off) + the Route Handler (SSE deliver)* (the module-33's scenario 6's pattern — the *slow compute* (module 18's external) — the *module-33's standing line: the slow compute → the background job + the SSE* (module 23-04's + module 08's) — the *no action for the slow*).

**The action** (the *module-29's five steps* — the *kick off*):

```ts
// src/features/catalog/product-actions.ts (the [SERVER] — the module-29's template, the *kick off*):
'use server'
// …imports (the module-29's template)
export async function startDescriptionGeneration(productId: string): Promise<void> {
  // 1 — the SESSION gate (the module-29's step 1):
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })
  const orgId = session.organizationId

  // 2 — the VALIDATION (the module-29's step 2):
  const id = z.string().uuid().parse(productId)

  // 2.5 — the RATE LIMIT (module 19's — the per-user — module 11's tenancy — the module-33's security note: the AI's rate limit is the action's (module 19's)):
  await assertRateLimit(session.user.id, 'ai-generate', 10, 60)   // the module-19's rate limit (the 10/min per user — the module-18's external's cost (module 18's) — the *module-19's line: the per-user rate limit* (module 11's tenancy)

  // 3 — the SERVICE (the module-29's step 3 — the *kick off* — the module-23-04's background job (Phase 23) — the *module-17's service* (the enqueue)):
  await enqueueDescriptionGeneration(orgId, id)   // the module-23-04's background job (Phase 23) — the *module-17's service* (the enqueue — the *no AI call here* (module 18's line: the external's cost — the *the* *the the *the AI call is the *job's* (module 23-04's) — the *action's* *no AI call* (module 18's line))

  // 4 — the INVALIDATION (the module-29's step 4 — the *no tag* (module 20's five questions: the WHO invalidates is no one (module 25's NOT-CACHED row) — the generation is the compute (module 24's))):
  // (no invalidation — the generate is the compute (module 24's) — the no tag (module 20's))

  // 5 — the RESPONSE (the module-29's step 5 — the redirect to the "generating" page (the module-09-04's redirect) — the *the* *the the *the SSE stream is the *Route Handler's* (module 08's) — the *module-33's scenario 7's pattern*):
  redirect(`/products/${id}/edit?generating=1`)   // the module-09-04's redirect (the "generating" page — the SSE stream (module 08's))
}
```

**The Route Handler** (the *SSE deliver* — the module-08's streaming response):

```ts
// src/app/api/descriptions/[id]/stream/route.ts (the [SERVER] — the module-08's SSE):
import { z } from 'zod'
import { headers } from 'next/headers'
import { auth } from '@/lib/auth'
import { getDescriptionStream } from '@/services/descriptions'   // the module-17's service (the *SSE stream* — the module-23-04's background job's *result stream*)

export async function GET(
  _request: Request,
  { params }: { params: Promise<{ id: string }> },
) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) {
    return new Response('Unauthorized', { status: 401 })
  }
  const { id } = await params
  if (!z.string().uuid().safeParse(id).success) {
    return new Response('Invalid ID', { status: 422 })
  }

  // the module-08's SSE (the module-33's scenario 7's pattern — the *text/event-stream*):
  const stream = getDescriptionStream(session.organizationId, id)   // the module-17's service (the *SSE stream* — the module-23-04's background job's result)
  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',   // the module-20's L2: the *no cache* (module 20's — the generate is the compute (module 24's))
    },
  })
}
```

**The client** (the *SSE consumer* — the module-13's island):

```tsx
// src/features/catalog/components/description-generator.tsx (the [CLIENT] island — the module-13's):
'use client'
import { useEffect, useRef, useState } from 'react'
import { startDescriptionGeneration } from '../product-actions'   // the module-29's action (the kick off)
import { useTransition } from 'react'

export function DescriptionGenerator({ productId }: { productId: string }) {
  const [isPending, startTransition] = useTransition()
  const [generated, setGenerated] = useState('')
  const eventSourceRef = useRef<EventSource | null>(null)

  const onGenerate = () => {
    startTransition(() => {
      void startDescriptionGeneration(productId)   // the module-29's action (the kick off — the module-33's scenario 6's pattern)
    })
    // the SSE (the module-08's) — the *module-33's scenario 7's pattern*:
    if (eventSourceRef.current) eventSourceRef.current.close()
    const es = new EventSource(`/api/descriptions/${productId}/stream`)
    eventSourceRef.current = es
    es.onmessage = (e) => {
      const text = e.data   // the module-14's DTO (the plain string — the module-14's serialization)
      setGenerated((prev) => prev + text)   // the *streaming* (the module-27's — the *text arrives incrementally*)
      if (e.data === '[DONE]') es.close()
    }
    es.onerror = () => es.close()
  }

  useEffect(() => () => { eventSourceRef.current?.close() }, [])   // the module-13's cleanup (the no leak)

  return (
    <div>
      <button onClick={onGenerate} disabled={isPending}>
        {isPending ? 'Generating…' : 'Generate description'}
      </button>
      {generated && <textarea value={generated} readOnly className="…" />}   // the module-31's form's field (the *the* *the the *the generated text is the form's field* (module 31's) — the *the user's edit is the form's* (module 31's) — the *no separate state* (module 13's line))
    </div>
  )
}
```

**The why** (the *three* decisions, the *module's* lines):
1. **The slow generate → the background job + the SSE** (the module-18's line: the external's cost — the module-33's standing line: the slow compute → the background job (module 23-04's) + the SSE (module 08's) — the *no action for the slow* (module 18's line: the 5–30s is the serverless's timeout (module 22-01's) — the module-33's scenario 6's pattern: the action kicks off the job + the Route Handler delivers) — the *module-33's line: the slow compute → the background job + the SSE* (module 23-04's + module 08's) — the *no action for the slow*).
2. **The AI's rate limit is the action's** (module 19's — the per-user — module 11's tenancy — the module-33's security note: the AI's rate limit is the action's (module 19's) — the no generate without a rate limit (module 19's) — the module-18's external's cost (module 18's) — the *module-19's line: the per-user rate limit* (module 11's tenancy)).
3. **The generated text is the form's field** (module 31's) — the *the user's edit is the form's* (module 31's) — the *no separate state* (module 13's line: the form owns the state) — the *module-33's line: the generated text is the form's field* (module 31's) — the *no separate state* (module 13's line).
**The generalization** (the *AI generate's* pattern, the *module's* standing rule): **a *slow compute* (the *5–30s* — the module-18's external) is *the action (kick off the job — module 23-04's) + the Route Handler (SSE deliver — module 08's)* — the *no action for the slow* (module 18's line: the external's cost — the serverless's timeout (module 22-01's)) — the *module-33's standing line: the slow compute → the background job + the SSE* (module 23-04's + module 08's) — the *no action for the slow* — the *the* *the the *the rate limit is the action's* (module 19's) — the *the result is the form's field* (module 31's) — the *module-33's scenario 6's pattern, the *AI's* version**.
</details>

## 10. Official Documentation

- Mutating Data (the Server Functions, the POST-only, the single roundtrip): https://nextjs.org/docs/app/getting-started/mutating-data
- Route Handlers (the module-08's — the `route.ts`, the HTTP methods, the streaming): https://nextjs.org/docs/app/building-your-application/routing/route-handlers
- `use server` (the directive, the file-level/inline): https://nextjs.org/docs/app/api-reference/directives/use-server
- Server Actions and Mutations (the security, the direct POST): https://nextjs.org/docs/app/guides/server-actions
- Forms (the action prop, the formAction): https://nextjs.org/docs/app/guides/forms
- Caching (the invalidation, the tags): https://nextjs.org/docs/app/getting-started/caching
- Revalidating (the `revalidateTag`, the webhook's pattern): https://nextjs.org/docs/app/getting-started/revalidating

## 11. What You Should Know Before Continuing (Phase 7 complete — the gate)

- [ ] I can state the *three doors* (the action, the Route Handler, the external API) — the *Q1/Q2/Q3* (module 2's decision procedure) — the *one-line procedure* (module 2's)
- [ ] I know the *invariant* (module 1's: the door is the transport; the service is the logic) — the *no self-API* (module 1's anti-pattern: the action calls the service — never its own Route Handler)
- [ ] I can *classify the 10 scenarios* (module 3's matrix) — the *Q1/Q2/Q3* per row — the *door* — the *one-line why*
- [ ] I know the *response shape overrides the caller* (module 2's Q2: the file/stream → the Route Handler — the *no action for the file/stream*)
- [ ] I know the *RSC re-render is the action's exclusive* (module 2's Q2: the RSC → the action — the *no Route Handler for the RSC*)
- [ ] I know the *external is the different scale/team* (Q3 — module 22-01's deployment physics) — the *no external for one internal feature* (module-33's common mistake #6)
- [ ] I know the *security lines* (the action's public route — module 29's §6; the Route Handler's API key — module 08-04's; the external's trust boundary — module 19's; the webhook's signature — module 23's §3.3)
- [ ] I've done the *decision audit* (module 8's production exercise: the four checks per door — the *Q1/Q2/Q3/service*) — the *artifact: the docs/surface-audit.md*
- [ ] **The phase gate (the roadmap):** "Chooses correctly for 10 scenarios" — *true, in the docs/surface-map.md*
- [ ] **Capstone stage after Phase 7 (the roadmap):** "Stage 5–8 mutation paths (products, orders, profile, settings)" — *true, in the docs/surface-map.md + the docs/action-inventory.md*

**Next phase:** Phase 8 — **Route Handlers & BFF** (the module-08's deep dive: the `route.ts` in depth — the HTTP methods, the response shapes (JSON/file/stream), the auth models (the session, the API key, the signature), the BFF's *legitimate* role (the module-16's "third party" — the *no self-API*), the *module-33's doors 2 and 3* in depth).
