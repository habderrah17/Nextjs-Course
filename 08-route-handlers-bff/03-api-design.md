# Module 36 — API Design & Response Contracts: The `/api/v1` the Partners Can Bet On

**Phase 8: Route Handlers & BFF · Module 36 of 101 · the final module of Phase 8**

> **Where does this run?** `[SERVER]` (the contract is *enforced* server-side, module 31's line) and *lives in a document* (`docs/api-v1.md`, module 35's §4). A response contract is a **promise to an external consumer** — the module-14's DTO discipline (the wire protocol) applied to *machines*, with the module-19's security and the module-18's performance as co-authors. This module is the capstone stage's deliverable: **the public API (products read) + the webhook receiver + the CSV export stream**, specified and consistent.

---

## 1. Concept — A contract is a *documented, versioned, tested* promise

The web's RSC payload is *internal* (module 35's §4 line: the web's contract is the *code* — module 17's DTO). A *public* API's contract is *external* — a partner writes code **against it today** that must work **six months from now**. That difference produces the contract's four properties:

1. **Documented** (module 35's gate): every endpoint — the method, the path, the query/body schema (Zod, module 04), the response shape, the *status codes with their meanings* (module 34's §3.2 table), the `Cache-Control`, the rate limits. *Undocumented behavior is undefined behavior* (the module-36's line: **if it's not in `docs/api-v1.md`, the partner can't rely on it**).
2. **Consistent** (this module's core): *one* envelope, *one* error format, *one* pagination, *one* naming style — across *every* endpoint. The partner learns the pattern *once* (the module-36's line: **consistency is the API's UX** — a surprise in endpoint #40 is the partner's ticket).
3. **Versioned** (§4): additive changes are *free* (the partner's code keeps working); breaking changes get a *version* (`/v2`) + a *deprecation* window (the `Sunset` header, §4.3).
4. **Tested** (the module-20's phase, previewed): the contract *is* the test suite's spec — a *contract test* per endpoint (the module-20's `vitest`: the handler's response *matches* the document — the module-36's line: **the document is the spec; the test is the proof**).

## 2. The Envelope (the module-36's core artifact)

### 2.1 Success: `{ data, meta? }`

```jsonc
// GET /api/v1/orgs/{orgId}/products?limit=2 — 200
{
  "data": [
    {
      "id": "9f1c…",              // the stable resource ID (the module-17's DTO — the UUID, never a DB serial)
      "slug": "aurora-stand",     // the stable human key (module 24's slug schema)
      "name": "Aurora Stand",
      "price_cents": 12900,       // INTEGER CENTS — the module-14's money rule, the wire version (module 35's §3's mobile contract, generalized: the PUBLIC wire uses integer cents, ALWAYS)
      "sku": "AST-001",
      "status": "active",         // the string enum (module 34's §2 — the documented set: 'active' | 'delisted' | 'draft')
      "images": ["https://cdn.….jpg"],
      "created_at": "2026-09-12T10:12:00Z",   // ISO-8601 UTC (module 14's date rule)
      "updated_at": "2026-09-20T08:00:00Z"
    }
    // …
  ],
  "meta": {
    "next_cursor": "eyJpZCI6IjlmMWM…"    // the module-40's cursor (opaque, base64 — the module-36's §3: the partner NEVER constructs it, only echoes it)
  }
}
```

**The envelope's rules** (module 36's, per key):
- **`data` is always present** — an array for collections, an object for single resources, *absent only in errors* (the module-36's line: the *shape is uniform* — the partner's parser: `body.data ?? throw` — *one* parser for all endpoints).
- **`meta` is optional, *pagination/async only*** (the cursor, the job status — module 34's challenge's `jobId`) — *never* for data the partner should act on (the module-36's line: **`meta` is the transport's concern (pagination/async); `data` is the domain's**).
- **`snake_case` on the wire** (module 35's §3's mobile contract, the public standard) — the *web's* RSC DTO is `camelCase` (module 17's) — the *two styles are the two consumers'* (module 35's two-DTO discipline) — the *wire is snake_case, the RSC is camelCase* (the module-36's line: **the wire's style is the contract's; the RSC's style is the code's**).
- **Money = integer cents, dates = ISO-8601 UTC, enums = documented strings, IDs = stable UUIDs** (the module-14's DTO rules, verbatim, on the wire) — *no floats for money, no locale dates, no numeric enums, no DB serials*.

### 2.2 Errors: `{ error: { status, code, message, fieldErrors?, details? } }`

The *single* error shape, for *every* non-2xx (module 34's §3.2's table, the *body* version):

```jsonc
// POST /api/v1/orgs/{orgId}/products — 422
{
  "error": {
    "status": 422,                          // = the HTTP status (the module-36's line: the body's status = the header's status — the partner checks ONE, not both)
    "code": "validation",                   // the STABLE machine code (the module-17's AppError code — module 34's §3.2's codes) — the partner's switch is on THIS (never the message)
    "message": "Invalid request body",      // the USER-FACING text (module 19-03's: no internals — the module-31's line, the wire version)
    "fieldErrors": {                        // OPTIONAL — 422s with per-field detail (module 31's mapping, the wire version)
      "price": ["price must be 0 or more"]
    },
    "details": [                            // OPTIONAL — the Zod issues' *stable* subset (module 04's — the `path` + `code`, NEVER the schema's shape)
      { "path": ["price"], "code": "too_small" }
    ]
  }
}
```

**The AppError → HTTP mapping** (module 17's taxonomy → the wire — the module-34's §3.2's table, the *complete* contract):

| AppError (module 17's) | HTTP | `code` | `fieldErrors`? | The partner's handling (documented in `docs/api-v1.md`) |
|---|---|---|---|---|
| `unauthenticated` | 401 | `unauthorized` | — | re-authenticate (re-fetch the key — module 36's §4.2's rotation) |
| `forbidden` (scope/org mismatch) | 403 | `forbidden` | — | the key lacks the scope / wrong org (module 33's §4) — *stop and surface* (no retry) |
| `bad_request` / `bad_json` | 400 | `bad-request` / `invalid-json` | — | the body is malformed — *developer bug* (no retry) |
| `validation` | 422 | `validation` | **yes** (module 31's) | fix the fields, resubmit (the `fieldErrors` drive the partner's form) |
| `not_found` | 404 | `not-found` | — | the resource is *gone or not in your org* (module 11-03's no-leak: *one* 404 for both — the partner can't distinguish *deleted* from *other org's* — the module-19's line) |
| `conflict` / `already-exists` | 409 | `conflict` / `already-exists` | optional (the field, module 31's 2b) | the state changed (module 09-04's idempotency: *re-read, then decide* — the module-34's §4's webhook's re-delivery, the partner's version) |
| `rate_limited` | 429 | `rate-limited` | — | back off per `Retry-After` (§5.2) — *the only 4xx the partner retries* |
| `unsupported_media_type` | 415 | `unsupported-media-type` | — | send `application/json` (module 34's §3.2) |
| *unexpected* (the module-17's 500) | 500 | `internal` | — | **the message is ALWAYS "Internal server error"** (module 19-03's, the wire's strictest line) — the `digest` (module 09-04's) is in the *logs* (module 21's) + the `x-request-id` header (§5.3) — the partner's retry: *only* idempotent methods (module 34's §2: the GET/PUT/DELETE) — *never* the POST without the §5.1 idempotency key |

**The `ApiErrorBody` helper** (module 34's §1's referenced function, defined *here* — the contract's enforcement point):

`FILE: src/lib/api-contract.ts` (production pattern — [SERVER])

```ts
// THE CONTRACT (module 36's §2 — the ONE error shape, the ONE helper — module 34's handlers import THIS, never build the body by hand):
import { z } from 'zod'

export const ApiErrorBodySchema = z.object({
  status: z.number(),
  code: z.string(),
  message: z.string(),
  fieldErrors: z.record(z.array(z.string())).optional(),
  details: z.array(z.object({ path: z.array(z.string()), code: z.string() })).optional(),
})
export type ApiErrorBody = { error: z.infer<typeof ApiErrorBodySchema> }

export function apiError(status: number, code: string, message: string, fieldErrors?: Record<string, string[]>): ApiErrorBody {
  // the module-19-03's line, enforced HERE (not per-handler): the 500's message is ALWAYS generic:
  const safeMessage = status === 500 ? 'Internal server error' : message
  return { error: { status, code, message: safeMessage, ...(fieldErrors && { fieldErrors }) } }
}

// THE SUCCESS ENVELOPE (module 36's §2.1 — the ONE shape):
export function apiData<T>(data: T, meta?: Record<string, unknown>) {
  return { data, ...(meta && { meta }) }
}
```

**The helper is the contract's *compiler*** (module 36's line): a handler that builds `{ error: 'oops' }` by hand is a *contract violation the reviewer must catch* — the helper makes the violation a *build error* (the TypeScript type — the module-04's line, the wire version: **the envelope is a type, not a convention**).

### 2.3 The resource set (the public API, the capstone stage — "products read")

`FILE: docs/api-v1.md` — the **public catalog** (the module-35's §4's table, the *products* endpoints, the capstone stage's deliverable):

```md
## Public catalog (no orgId — the module-35's §4's "the no orgId in the URL (the public)")
GET /api/v1/products                  → 200 { data: ProductV1[], meta: { next_cursor } }
     query: cursor?, limit? (1–100, default 20), status? (default 'active'), category?, q? (Phase 17's search)
     Cache-Control: s-maxage=60, stale-while-revalidate=300   (the module-20's L2 CDN — the public's GET is the CDN-cacheable (module 34's §2) — the module-36's §5.3's line: the public catalog is the CDN's)
     Rate: 600/min/key (module 36's §5.2)
GET /api/v1/products/{slug}           → 200 { data: ProductV1 }  ·  404 not-found (the delisted = 404, module 30's soft-delete's wire consequence: the delisted product is GONE from the public (module 34's §2's DELETE's idempotency) — the partner's 404 handling: the module-11-03's no-leak (deleted = other org's = delisted = ONE 404))
```

**The delisted=404 line** (module 36's, the soft-delete's wire consequence): the module-30's delist is a *status* (`'delisted'`) — the *public* API **filters** on `status='active'` by default (the module-34's §1's `querySchema`'s default) — a delisted product's *detail* returns **404** (not the 200-with-status — the module-19's line: *don't advertise "delisted" to the public* — the *partner's* inventory sync is the 404-driven (the module-36's line: **the public API's delist is a 404; the org-scoped API's delist is the 200-with-status** (the *org* sees the status (module 11's tenancy: it *owns* the product) — the *two truths, two doors* (module 33's) — the module-36's §2.3's line)).

## 3. Pagination — Cursor-based, always (the module-40's preview)

**The rule** (module 36's, the module-40's deep dive is Phase 9): the public API paginates **cursor-only** (the module-40's offset+cursor: the *wire* is cursor; the *app's internal* service supports both, module 40) — *no `page`/`offset` on the wire* (the module-36's line: **offset pagination on a mutating feed is the module-18's "the partner's list skips/duplicates rows" ticket** — the cursor is the *stable* (module 40's) — the §2.1's `meta.next_cursor`).

| Wire piece | The contract | The module-40's mechanics (preview) |
|---|---|---|
| `?limit=` | 1–100, default 20 (the module-34's §1's `z.coerce.number().min(1).max(100).default(20)`) | the `LIMIT` + the *over-fetch by 1* (the module-40's cursor's "is there a next" — the §3.2's) |
| `?cursor=` | *opaque* base64 (the module-2.1's) — the partner **echoes** it, never parses it | the encoded `(last_id, last_created_at)` (the module-40's composite cursor — the *stable sort* (module 40's: the `ORDER BY created_at DESC, id DESC` — the *no ties*) |
| `meta.next_cursor` | present iff there's a next page (the over-fetch, §3.2) — *absent* on the last page (the module-36's line: **the partner's loop: `while (cursor) { fetch; cursor = meta.next_cursor ?? stop }` — one loop, all endpoints**) | the *no `has_more`* (the *nullable cursor is the boolean* — the module-36's line: *one field, not two* (the `has_more` + the cursor can *disagree* — the *nullable cursor is the single source*)) |
| The *stable sort* | `created_at DESC, id DESC` (the module-40's) — the *documented* (`docs/api-v1.md`'s "Ordering" section) | the *no `ORDER BY rand()`* (module 40's — the cursor needs a *total order*) |

**§3.2 — the over-fetch by 1** (the cursor's "next page exists" test, module 40's, the wire's contract): the service queries `limit + 1` rows; if it gets `limit + 1`, *drop the last* and set `next_cursor = the last kept row's (id, created_at)`; if it gets `≤ limit`, `next_cursor = null`. **The wire never exposes the mechanism** (the cursor's *encoding* is an implementation — the module-36's line: **the cursor is opaque *on purpose*** (a versioned API that exposes the cursor's *internal* (the DB's `id`) leaks the schema (module 19-03's) + breaks on a re-index (module 40's) — the *base64(opaque)* is the contract; the *decoding* is the service's).

## 4. Auth for the public API (the module-36's §4 — the machine's identity)

### 4.1 The API key (the capstone's primary)

- **The header**: `x-api-key: sk_live_…` (module 34's §6: *the header, never the URL* — module 19's: the URL is logged (module 21's) — the key in the URL is the *leak*).
- **The format**: `sk_live_` / `sk_test_` + 32 random bytes, base62 (the module-19's line: the *prefix is the environment* — the *test key is REJECTED in prod* (module 19's: the environment check — the module-36's §4.1's line: **the test key in prod is a 401 with `code: 'wrong-environment'`** (the module-19-03's: no "your key is a test key" specifics beyond the code) — the *module-22's env's line* (module 02's 4-split env: the *test/prod* key prefixes are the env's check)).
- **The lookup**: the *hashed* key (the module-19's line: **the DB stores the SHA-256 hash, never the key** — module 10-03's password-hash line, the key version) → the `org_id` + the `scopes[]` + the `expires_at` (module 36's §4.2) — the *module-17's service* (`getOrgByApiKey`, module 33's §4) — the *cache*: the key lookup is a *module-21's `'use cache'` entry with a SHORT life (the module-22's `minutes` profile — the *module-36's line: the key lookup is cached (the module-20's L3) with a short life (the module-22's `minutes`) — the *revocation* is the `revalidateTag('api-key:' + keyId)` (module 23's) — the *module-36's §4.2's revocation: the key's row is updated + the tag is revalidated (module 23's) — the *≤ 5min* to take effect (the module-22's `minutes`'s stale — the *documented* in `docs/api-v1.md`: "revocation takes effect within 5 minutes" (the *module-36's line: the *documented* revocation window (module 35's gate) — the *no "instant" promise* (the module-22's `minutes`'s stale — the *honest*)*)**.
- **The scopes** (module 11's RBAC, the wire version): `products:read`, `products:write`, `orders:read`, `orders:write`, `settings:write` — the *key is created with the scopes* (module 36's §4.2's admin action, *in the app* — the org owner's UI, an *action* (module 29's) — the *key management UI is the APP's* (module 35's challenge's portal is the *public API's* developer portal — *different* (module 33's Q3) — the *org's* key management is the *app's* action (module 29's) — the *module-36's line: the org's keys are the app's actions; the public API's developer accounts are the portal's (module 35's challenge) — the two key systems, two owners* (module 33's Q3)).
- **The tenancy check** (module 33's §4, module 34's): the *key's org* vs the *URL's org* — the mismatch = the 403 `forbidden` (module 34's §4) — the *public catalog* (`/products`) has *no orgId* (module 35's §4's line: the no orgId in the URL) — the *key still required* (the rate limit's identity, §5.2 — the *module-36's line: the public catalog is *authenticated* (the key) but *not org-scoped* (the no orgId) — the *the* *the the *the public ≠ the anonymous* (module 19's: the *anonymous* is the *rate limit's* *IP* (module 19's) — the *keyed* is the *rate limit's* *key* (module 36's) — the *module-36's line: the public catalog requires the key* (module 19's) — the *no anonymous public API* (module 19's DoS line*)**.

### 4.2 The key lifecycle (the module-19's rotation, the app's actions)

`FILE: src/features/api-keys/api-key-actions.ts` (production pattern — [SERVER] — the *app's* action, module 29's five steps):

```ts
'use server'
// createApiKey(formData):  the session's org (module 29's step 1) → the scopes' Zod (module 31's 2a) →
//   createApiKeyService(orgId, { scopes, label })   (module 17's: the key = crypto.randomBytes(32) base62;
//     the DB stores the SHA-256 HASH + the org + the scopes + the expires_at (module 19's);
//     the PLAIN key is returned ONCE (the module-19's line: the *the* *the the *the plain key is shown once, never stored* (module 19-03's) — the *module-36's line: the *create's response shows the key once* (module 19's) — the *the* *the the *the action's return is the DTO* (module 29's §4's exception) — the *the* *the the *the key's display is the *modal* (module 13's) — the *the* *the the *the "copy now, it's gone" (module 13's copy)
// revokeApiKey(keyId):     the session's org → the *can't revoke the last active key*? (module 11's line: the *no* (module 11-03's: the *revoke is allowed* — the *org without keys* = the *partner locked out* (module 11's RBAC's line: the *owner can always create* (module 11's) — the *module-36's line: the *revoke is always allowed* (module 11's) — the *the* *the the *the last-key check is the *portal's* (module 35's challenge) — the *app's* org: the *revoke is allowed* (module 36's) — the *the* *the the *the module-36's line: the *org's* key revoke is allowed (module 11's) — the *portal's* developer key: the *no last-key* (module 35's challenge) — the *two systems*) →
//   revokeApiKeyService(orgId, keyId)   (module 17's: the expires_at = now + the revalidateTag('api-key:' + keyId) (module 23's — the §4.1's revocation window)
// rotateApiKey(keyId):     the create + the revoke in a transaction (module 09-04's: the *old hash's* `expires_at` = now + 24h (the module-19's line: the *rotation's grace window* (module 19's) — the *old key works 24h* (the *documented*) — the *module-36's line: the *rotation is the create + the 24h-grace-revoke* (module 19's) — the *the* *the the *the module-19's rotation: the *dual-key window* (module 19's))
```

### 4.3 Versioning (the module-36's §4's line — the partner's bet)

- **The URL version**: `/api/v1/…` (module 35's §4) — the *v1 is the capstone's* (module 36's).
- **Additive = free** (the module-36's line): a *new field* in `data` (the `ProductV1.brand`), a *new endpoint*, a *new query param* (defaulted) — the partner's code *ignores* the new (the *module-36's line: the *additive is the partner's no-op* (module 35's contract: the *documented* "fields may be added" (module 36's §1's documented property) — the *module-36's line: the *additive is the v1's* (module 36's) — the *no version bump* (module 36's)).
- **Breaking = a version** (the module-36's line): a *renamed field*, a *changed type* (the `price_cents` → the `price` dollars), a *removed field*, a *changed status code's meaning* — the **`/v2`** (module 36's) — the *v1 keeps running* (the module-36's line: the *v1's sunset is the *documented* window (module 36's §4.3's line: the *v1 runs until the Sunset date* (module 36's) — the *module-19's line: the *v1's traffic is the *v1's* (module 18's) — the *no silent kill* (module 36's)).
- **The `Sunset` header** (the module-36's §4.3's RFC 8594): on the *deprecating* endpoint: `Sunset: Sat, 01 Mar 2027 00:00:00 GMT` + the `Deprecation: true` + the `Link: <https://docs.….v2>; rel="successor-version"` — the *documented* in `docs/api-v1.md`'s "Deprecation" section (module 35's gate) — the *module-36's line: the *deprecation is the *header + the doc* (module 36's §4.3) — the *no silent deprecation* (module 36's)*.

## 5. Rate limiting, idempotency, and the request ID (the module-36's §5 — the partner's operational contract)

### 5.1 Idempotency keys (the POST's re-delivery, module 34's §4's webhook line, the partner's version)

The *partner's* POST (the `create product`) *times out* (the module-19's line: the *network* — the *the* *the the *the partner's retry is the *module-09-04's* *double-create* (module 29's create is *not idempotent*) — the *module-36's line: the *POST's idempotency is the *Idempotency-Key header* (module 36's §5.1) — the *module-09-04's idempotency is the *service's* (module 17's) — the *the* *the the *the module-36's line: the *Idempotency-Key is the *partner's* *re-delivery protection* (module 36's) — the *module-09-04's guard is the *app's* (module 17's) — the *two are separate* (module 36's §5.1 vs module 09-04's guard)*.

```
POST /api/v1/orgs/{orgId}/products
Idempotency-Key: 7f3a-… (the partner's UUID — the module-19's line: the *the partner generates it* (module 19's) — the *module-36's line: the *Idempotency-Key is the *partner's* (module 36's) — the *the* *the the *the no server-generated key* (module 36's: the *the partner needs it for the *retry* (module 19's) — the *the* *the the *the module-36's line: the *Idempotency-Key is the *partner's UUID* (module 36's) — the *the* *the the *the no server-generated*)
```

**The mechanics** (module 36's §5.1, the module-17's service): the `(orgId, idempotencyKey)` → a *stored* row (the *key + the response hash + the created resource's id* — the module-17's `idempotency_keys` table, module 9's Phase 9) — the *first* POST: the *store the key* + the *process* + the *store the response* (the module-09-04's transaction) — the *retry* (the *same key*): the *return the stored response* (the *no re-process* — the *module-36's line: the *retry is the *stored response* (module 36's §5.1) — the *no double-create* (module 29's create is *not idempotent* — the *Idempotency-Key is the *protection*)*) — the *key's life*: the *24h* (the module-19's line: the *idempotency key's window* (module 19's) — the *documented* in `docs/api-v1.md`) — the *module-36's line: the *Idempotency-Key's window is the *24h* (module 19's) — the *documented* (module 35's gate)*.

### 5.2 Rate limits (the module-19's line, the wire's contract)

- **The limits** (the module-36's §5.2's table, the *documented*): the *public catalog*: 600/min/key; the *org-scoped reads*: 120/min/key; the *mutations*: 60/min/key; the *bulk import*: 10/hour/key (module 34's challenge) — the *module-19's line: the *limits are the *documented* (module 35's gate) — the *no undocumented limits* (module 19's) — the *module-36's line: the *rate limits are the *documented* (module 35's gate) — the *no undocumented* (module 19's)*.
- **The headers** (the module-36's §5.2's, the *RFC 6585's* + the *de-facto standard*): on *every* response: `X-RateLimit-Limit: 600` · `X-RateLimit-Remaining: 599` · `X-RateLimit-Reset: 1727500000` (the epoch) — on the *429*: `Retry-After: 30` (the *seconds* — the module-36's line: the *Retry-After is the *partner's backoff* (module 36's §5.2) — the *the* *the the *the module-19's line: the *429 is the *only 4xx the partner retries* (module 34's §3.2's table) — the *Retry-After is the *backoff* (module 36's §5.2)*.
- **The limiter** (the module-19's, the module-17's service): the *fixed window* (the module-19's line: the *simple* — the *module-18's* *cost* (module 18's) — the *module-36's line: the *fixed window is the *capstone's* (module 19's) — the *token bucket is the *scale's* (module 18's) — the *module-22-01's* *serverless* (module 22-01's: the *in-memory fixed window* (module 22-01's) — the *the* *the the *the module-36's line: the *rate limit is the *in-memory fixed window* (module 19's) for the *capstone* (module 22-01's) — the *Redis is the *scale's* (module 22-01's) — the *module-18's* *cost* (module 18's))* — the *the* *the the *the module-19's line: the *rate limit is the *service's* (module 17's) — the *no handler-level limiter* (module 17's) — the *module-36's line: the *rate limit is the *service's* (module 17's) — the *no handler-level* (module 17's)*.

### 5.3 The `x-request-id` (the module-21's observability, the wire's contract)

On *every* response: `x-request-id: <uuid>` (the module-21's trace, module 19's tracer — the *module-36's line: the *x-request-id is the *partner's* *support ticket* (module 36's §5.3) — the *the* *the the *the module-21's line: the *x-request-id is the *observability's* (module 21's) — the *the* *the the *the module-36's line: the *x-request-id is the *partner's* *support ticket* (module 36's) — the *the partner's "my request failed" is the *x-request-id* (module 21's) — the *the* *the the *the module-19-03's line: the *500's body is generic* (module 34's §6) — the *x-request-id is the *detail* (module 21's) — the *the* *the the *the module-36's line: the *500's detail is the *x-request-id* (module 21's) — the *no detail in the body* (module 19-03's)*).

## 6. The CSV export stream (the capstone stage's third deliverable — module 33's scenario 6)

`FILE: src/app/api/v1/orgs/[orgId]/reports/[jobId]/download/route.ts` (production pattern — [SERVER] — the module-33's scenario 6's *deliver* half):

```ts
// THE ACTION (the kick-off — module 29's, in the app's UI):
//   exportOrdersCsv(formData): the session (module 29's step 1) → the filters' Zod (module 31's) →
//     enqueueReportJob(orgId, { type: 'orders-csv', filters }) (module 23-04's job — Phase 23) →
//     redirect(`/settings/reports?job=${jobId}`)   (the module-09-04's redirect — the *the report's page* (module 27's hole: the *status* (module 34's challenge's `jobId`))
//   THE JOB (module 23-04's, Phase 23): the streaming write (module 34's §3.3's (b) — the *no full in-memory* (module 18's line) — the *S3/Cloudinary* (module 16's) — the *the report's URL* (module 17's DTO))
//   THE DELIVER (this file): the *stream* from the storage (module 16's) — the *auth: the session* (module 29's step 1 — the *org's* report (module 11's tenancy) — the *no public* (module 19's: the *report is the *org's data* (module 11's) — the *no public download* (module 19's))):

export async function GET(request: Request, { params }: { params: Promise<{ orgId: string; jobId: string }> }) {
  const session = await auth.api.getSession({ headers: await headers() })   // the module-10's session (the *org's* member — module 29's step 1)
  if (!session?.organizationId) return new Response('Unauthorized', { status: 401 })
  const { orgId, jobId } = await params
  if (session.organizationId !== orgId) return new Response('Forbidden', { status: 403 })   // the module-11-03's 403 (the *wrong org*)

  const report = await getReportFile(session.organizationId, jobId)   // the module-17's service (the *tenancy* (module 17's rule 1) — the *the report's* *file URL* (module 16's))
  if (!report) return new Response('Not found', { status: 404 })

  // the module-34's §3.3's (b) — the *stream* (module 18's line: the *no full in-memory*) — the *module-16's storage* (module 16's) — the *module-18's* *cost* (module 18's):
  const source = await fetch(report.fileUrl)   // the module-16's storage (module 16's)
  return new Response(source.body, {
    headers: {
      'Content-Type': 'text/csv',
      'Content-Disposition': `attachment; filename="${report.filename}"`,
      'Cache-Control': 'no-store',   // the module-36's §5.3's line: the *report is the *org's data* (module 11's) — the *no cache* (module 20's)
    },
  })
}
```

**The capstone stage, complete** (the roadmap's "Public API (products read) + webhook receiver + CSV export stream"):
1. **The public API (products read)** — module 36's §2.3's endpoints + the §2's envelope + the §3's cursor + the §4.1's key + the §5.2's rate limit — the *documented* in `docs/api-v1.md` (module 35's gate) — the *module-36's line: the *public API is the *documented* (module 35's gate) — the *no undocumented* (module 19's)*.
2. **The webhook receiver** — module 34's §4's full form (the signature FIRST, the 2xx-fast, the idempotency (module 09-04's guard), the `revalidateTag` (module 23's)) — the *documented* (module 35's gate) — the *module-36's line: the *webhook is the *documented* (module 35's gate) — the *no undocumented* (module 19's)*.
3. **The CSV export stream** — the module-33's scenario 6's two-step (the action's kick-off + the Route Handler's deliver) — the *stream* (module 34's §3.3's (b) — the *no full in-memory* (module 18's line)) — the *documented* (module 35's gate) — the *module-36's line: the *CSV is the *documented* (module 35's gate) — the *no undocumented* (module 19's)*.

## 7. Common Mistakes (the contract failures)

| Mistake | The partner's ticket | Fix |
|---|---|---|
| **The inconsistent envelope** (the endpoint #7's `return NextResponse.json({ products: […] })` — the *no `data`* (module 2.1's line: the *`data` is always present*)) | The *partner's parser breaks* (the `body.data ?? throw` — the module-2.1's line) — the *module-36's line: consistency is the API's UX* (module 1's line) | The *`apiData` helper* (module 2.2's) — the *ONE shape* (module 2.1's) — the *no by-hand envelope* (module 2.2's line: the *envelope is a type* (module 04's line, the wire version)) |
| **The inconsistent error** (the endpoint #9's `return NextResponse.json({ error: 'not found' }, { status: 404 })` — the *no `code`* (module 2.2's: the *`code` is the stable machine code*)) | The *partner's switch breaks* (the `switch (body.error.code)` — the module-2.2's line) — the *module-36's line: the *code is the *partner's switch* (module 2.2's) — the *no string-matching the *message* (module 19-03's: the *message is the *user-facing* (module 31's) — the *the* *the the *the module-36's line: the *partner's switch is the *code* (module 2.2's) — the *no message-matching* (module 19-03's)* | The *`apiError` helper* (module 2.2's) — the *ONE shape* (module 2.2's) — the *the AppError → the HTTP mapping* (module 2.2's table) — the *no by-hand error* (module 2.2's line) |
| **The offset pagination on the wire** (module 3's line violated: the `?page=`) | The *partner's list skips/duplicates* (the module-18's "the rows move" — module 3's line) — the *module-40's* *cursor* (module 40's) — the *module-36's line: the *wire is cursor* (module 3's) — the *no offset* (module 40's) | The *cursor* (module 3's) — the *module-40's* *mechanics* (module 40's) — the *the* *the the *the module-36's line: the *wire is cursor* (module 3's) — the *no offset* (module 40's)* |
| **The float money** (module 2.1's line violated: the `price: 12.99`) | The *partner's rounding bug* (the `12.99 * 100 ≠ 1299` (module 14's money line) — the *module-36's line: the *money is the *integer cents* (module 2.1's) — the *no float* (module 14's) | The *`price_cents`* (module 2.1's) — the *module-14's* *money line* (module 14's) — the *module-36's line: the *money is the *integer cents* (module 2.1's) — the *no float* (module 14's)* |
| **The key in the URL** (module 4.1's line violated: the `?key=sk_live_…`) | The *leak* (module 19's: the URL is logged (module 21's) — the *module-36's line: the *key is the *header* (module 4.1's) — the *no URL* (module 19's) | The *`x-api-key` header* (module 4.1's) — the *module-19's line: the *key is the *header* (module 4.1's) — the *no URL* (module 19's)* |
| **The plaintext key in the DB** (module 4.1's line violated: the *key stored, not hashed*)) | The *breach* (module 19's: the *DB's breach* → the *all the keys* (module 19-03's) — the *module-36's line: the *key is the *hashed* (module 4.1's) — the *no plaintext* (module 19's) | The *SHA-256 hash* (module 4.1's) — the *module-19's line: the *key is the *hashed* (module 4.1's) — the *no plaintext* (module 19's)* |
| **The silent deprecation** (module 4.3's line violated: the *v1's field removed, no `/v2`, no `Sunset`)) | The *partner's outage* (module 4.3's line: the *breaking is a version* (module 4.3's) — the *module-36's line: the *deprecation is the *header + the doc* (module 4.3's) — the *no silent* (module 36's) | The *`/v2`* + the *`Sunset` header* (module 4.3's) + the *doc* (module 35's gate) — the *module-36's line: the *deprecation is the *header + the doc* (module 4.3's) — the *no silent* (module 36's)* |
| **The undocumented rate limit** (module 5.2's line violated: the *429 with no `Retry-After`*)) | The *partner's retry storm* (module 19's: the *no backoff* → the *thundering herd* (module 22's storm) — the *module-36's line: the *rate limit is the *documented* (module 35's gate) — the *no undocumented* (module 19's) | The *`X-RateLimit-*` + the `Retry-After`* (module 5.2's) + the *doc* (module 35's gate) — the *module-36's line: the *rate limit is the *documented* (module 35's gate) — the *no undocumented* (module 19's)* |

## 8. Security & Performance Notes

- **The security is the module-19's** (module 4's: the *key is the header*, the *hashed*, the *scopes*, the *tenancy*, the *rotation* (module 19's) — the *module-36's line: the *security is the module-19's* (module 4's) — the *no contract without security* (module 19's)*.
- **The performance is the module-18's** (module 3's: the *cursor's* *stable sort* (module 40's) — the *module-18's* *cost* (module 18's) — the *module-36's line: the *performance is the module-18's* (module 3's) — the *no contract without performance* (module 18's)*.
- **The CDN is the module-20's L2** (module 2.3's: the *public catalog is the CDN-cacheable* (module 34's §2) — the *module-36's line: the *CDN is the module-20's L2* (module 2.3's) — the *no public API without a CDN* (module 20's)*.

## 9. Exercise

**Beginner.** *The envelope + the error* (module 2's): the *`src/lib/api-contract.ts`* (module 2.2's) — the *`apiData`* + the *`apiError`* — the *the AppError → the HTTP mapping* (module 2.2's table) — *build it* — the *test* (module 20's: the *`apiError(500, 'internal', '…')`* → the *message is the *generic* (module 19-03's) — the *module-36's line: the *500's message is the *generic* (module 2.2's) — the *no internals* (module 19-03's)) — the *artifact: the `vitest`'s output (module 20's)*.

**Intermediate.** *The public catalog* (module 2.3's): the *`GET /api/v1/products`* + the *`GET /api/v1/products/{slug}`* — the *envelope* (module 2.1's) + the *cursor* (module 3's) + the *key* (module 4.1's) + the *rate limit* (module 5.2's) + the *`x-request-id`* (module 5.3's) — *build it* — the *curl* (the *key* + the *query* + the *cursor's echo*) — the *test* (module 20's: the *cursor's loop* (module 3's: the `while (cursor)`) — the *the 404 delisted* (module 2.3's line) — the *the 429* (module 5.2's: the *the rate limit*)) — the *artifact: the curl's outputs (module 20's)*.

**Production.** *The contract test suite* (module 1's line: the *document is the spec; the test is the proof*): the *`docs/api-v1.md`* (module 4's) + the *`vitest`'s contract tests* (module 20's: the *per-endpoint* — the *response matches the doc* (module 1's line)) — the *the* *the the *the module-36's line: the *contract test is the *doc's proof* (module 1's) — the *no doc without a test* (module 1's) — the *the* *the the *the module-20's* *phase's* *preview* (module 20's) — the *module-36's line: the *contract test is the *doc's proof* (module 1's) — the *no doc without a test* (module 1's)* — the *artifact: the `vitest`'s contract suite (module 20's)*.

## 10. Architecture Challenge

**Prompt:** The *"the partner complains: 'your API is too slow for our mobile app'"* (the *module-36's* *public API* — the *mobile's* *list view* — the *module-18's* *performance* (module 18's) — the *module-36's line: the *partner's complaint is the *module-18's* *performance* (module 18's) — the *the* *the the *the module-36's line: the *partner's complaint is the *module-18's* (module 18's) — the *the* *the the *the module-36's line: the *partner's complaint is the *module-18's* (module 18's) — the *the* *the the *the module-18's* *line: the *measure first* (module 18-01's) — the *no guessing* (module 18's)*.

The *problems*: (1) the *measure* (module 18-01's: the *RUM* (module 21's) + the *server's* *TTFB* (module 18-01's) — the *module-36's line: the *partner's complaint is the *measure* (module 18-01's) — the *no guessing* (module 18's) — the *module-18's* *line: the *measure first* (module 18-01's) — the *no guessing* (module 18's)*.

(2) the *CDN* (module 20's L2: the *public catalog is the CDN-cacheable* (module 2.3's) — the *module-36's line: the *CDN is the module-20's L2* (module 2.3's) — the *no public API without a CDN* (module 20's) — the *the* *the the *the module-20's* *line: the *CDN is the *L2* (module 20's) — the *module-36's line: the *CDN is the module-20's L2* (module 2.3's) — the *no public API without a CDN* (module 20's)*.

(3) the *pagination* (module 3's: the *cursor's* *limit* (module 3's: the *`limit=100`* — the *module-18's* *cost* (module 18's) — the *module-36's line: the *pagination is the module-3's* (module 3's) — the *the* *the the *the module-3's* *line: the *cursor is the *stable* (module 40's) — the *module-36's line: the *pagination is the module-3's* (module 3's) — the *the* *the the *the module-18's* *line: the *limit is the *cost* (module 18's) — the *module-36's line: the *pagination is the module-3's* (module 3's) — the *the* *the the *the module-18's* *line: the *limit is the *cost* (module 18's)*.

**Design**: the *partner's complaint* (the *module-18's* *measure* (module 18-01's) + the *CDN* (module 20's L2) + the *pagination* (module 3's) — the *module-36's line: the *partner's complaint is the *module-18's* *measure* + the *CDN* + the *pagination* (module 3's) — the *module-36's standing line: the *partner's performance complaint is the *module-18's* *measure* (module 18-01's) + the *CDN* (module 20's L2) + the *pagination* (module 3's) — the *no guessing* (module 18's)*.

Produce: the *measure* (module 18-01's: the *RUM* (module 21's) + the *TTFB* (module 18-01's)) — the *CDN* (module 20's L2: the *`s-maxage`* (module 2.3's) — the *the* *the the *the module-20's* *line: the *CDN is the *L2* (module 20's)) — the *pagination* (module 3's: the *`limit`* (module 3's) — the *module-18's* *cost* (module 18's)) — and the *module-36's standing line: the *partner's performance complaint is the *module-18's* *measure* (module 18-01's) + the *CDN* (module 20's L2) + the *pagination* (module 3's) — the *no guessing* (module 18's)*.

<details>
<summary>Model answer</summary>
**The measure** (module 18-01's: the *no guessing* (module 18's)):
- The *RUM* (module 21's): the *partner's mobile app's* *TTFB* (module 18-01's) — the *module-21's* *analytics* (module 21's) — the *the* *the the *the module-21's* *line: the *RUM is the *user's* (module 21's) — the *module-36's line: the *RUM is the *partner's* (module 21's) — the *the* *the the *the module-21's* *line: the *RUM is the *user's* (module 21's) — the *module-36's line: the *RUM is the *partner's* (module 21's)*.
- The *server's TTFB* (module 18-01's): the *app's* *server's* *TTFB* (module 18-01's) — the *module-19's* *tracer* (module 19's) — the *the* *the the *the module-19's* *line: the *tracer is the *server's* (module 19's) — the *module-36's line: the *tracer is the *server's* (module 19's) — the *the* *the the *the module-19's* *line: the *tracer is the *server's* (module 19's)*.
- The *module-18-01's* *line: the *measure first* (module 18-01's) — the *no guessing* (module 18's) — the *module-36's line: the *measure first* (module 18-01's) — the *no guessing* (module 18's).

**The CDN** (module 20's L2: the *`s-maxage=60, stale-while-revalidate=300`* (module 2.3's)):
- The *module-20's* *line: the *CDN is the *L2* (module 20's) — the *module-36's line: the *CDN is the module-20's L2* (module 2.3's) — the *no public API without a CDN* (module 20's).
- The *the* *the the *the module-20's* *line: the *CDN is the *L2* (module 20's) — the *module-36's line: the *CDN is the module-20's L2* (module 2.3's) — the *no public API without a CDN* (module 20's).

**The pagination** (module 3's: the *`limit=100`* — the *module-18's* *cost* (module 18's)):
- The *module-3's* *line: the *cursor is the *stable* (module 40's) — the *module-36's line: the *pagination is the module-3's* (module 3's).
- The *module-18's* *line: the *limit is the *cost* (module 18's) — the *module-36's line: the *limit is the *cost* (module 18's).

**The module-36's standing line**: the *partner's performance complaint is the *module-18's* *measure* (module 18-01's) + the *CDN* (module 20's L2) + the *pagination* (module 3's) — the *no guessing* (module 18's).
</details>

## 11. Official Documentation

- Route Handlers (the contract's enforcement — module 34's): https://nextjs.org/docs/app/building-your-application/routing/route-handlers
- Caching (the CDN, the L2 — module 20's): https://nextjs.org/docs/app/getting-started/caching
- The `Sunset` header (RFC 8594 — module 4.3's): https://datatracker.ietf.org/doc/html/rfc8594
- Rate limiting (the `Retry-After`, RFC 6585 — module 5.2's): https://datatracker.ietf.org/doc/html/rfc6585
- OpenAPI (the contract's document format — the course's `docs/api-v1.md`, module 35's gate): https://www.openapis.org/
- The Zod (the contract's schema — module 04's): https://zod.dev/

## 12. What You Should Know Before Continuing (Phase 8 complete — the gate)

- [ ] I can state the *contract's 4 properties* (module 1's: the *documented*, the *consistent*, the *versioned*, the *tested*) — the *module-36's line: the *contract is the *documented, consistent, versioned, tested* (module 1's) — the *no undocumented* (module 19's)*
- [ ] I know the *envelope* (module 2's: the *`{ data, meta? }`* — the *`data` is always present* — the *`meta` is the *transport's* (module 2.1's) — the *snake_case* (module 2.1's) — the *integer cents* (module 2.1's) — the *ISO-8601* (module 2.1's))
- [ ] I know the *error format* (module 2.2's: the *`{ error: { status, code, message, fieldErrors?, details? } }`* — the *AppError → the HTTP mapping* (module 2.2's table) — the *the `apiError` helper* (module 2.2's) — the *the 500's message is the *generic* (module 19-03's))
- [ ] I know the *pagination* (module 3's: the *cursor-only* — the *opaque* (module 3.2's) — the *the over-fetch by 1* (module 3.2's) — the *the stable sort* (module 40's) — the *no offset* (module 40's))
- [ ] I know the *auth* (module 4's: the *key is the header* (module 4.1's) — the *hashed* (module 4.1's) — the *scopes* (module 4.1's) — the *rotation* (module 4.2's) — the *versioning* (module 4.3's: the *`Sunset`* (module 4.3's))
- [ ] I know the *rate limit* (module 5.2's: the *`X-RateLimit-*`* + the *`Retry-After`* — the *the 429 is the *only 4xx the partner retries* (module 34's §3.2's table) — the *documented* (module 35's gate))
- [ ] I know the *idempotency* (module 5.1's: the *`Idempotency-Key`* — the *the partner's UUID* (module 5.1's) — the *the 24h window* (module 19's) — the *documented* (module 35's gate))
- [ ] I know the *`x-request-id`* (module 5.3's: the *the partner's support ticket* (module 5.3's) — the *the 500's detail is the *x-request-id* (module 21's) — the *no detail in the body* (module 19-03's))
- [ ] **The capstone stage (the roadmap):** "Public API (products read) + webhook receiver + CSV export stream" — *true, in the docs/api-v1.md + the module-34's §4's webhook + the module-6's CSV*
- [ ] I've done the *envelope + the error* (module 9's beginner) + the *public catalog* (module 9's intermediate) + the *contract test suite* (module 9's production) — the *artifacts: the `vitest`'s outputs*

**Next phase:** Phase 9 — **Database & ORM** (the module-09's deep dive: the Postgres + the Drizzle setup (module 37's) — the *schema* + the *migrations* (module 38's) — the *queries* + the *relations* (module 39's) — the *transactions* + the *pagination* + the *indexes* (module 40's) — the *connections* + the *serverless pooling* (module 41's) — the *Prisma alternative* (module 42's)).
