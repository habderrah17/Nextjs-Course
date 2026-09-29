# Module 30 — Forms & Progressive Enhancement: The Form That Works Without JavaScript

**Phase 7: Server Functions / Actions · Module 30 of 101**

> **Where does this run?** The `<form>` is **`[BOTH — BOUNDARY]`**: it *renders* in the server tree (`[SERVER]` — its `action` is a Server Function reference), it *submits* from the client (`[CLIENT]`) — and crucially, it *works with the client absent* (the **progressive enhancement**: JS disabled → the native `POST` → the action runs → the 303 → the fresh page). The form is the only UI element where "works without JS" is *the default behavior of the framework*, not a feature you build.

---

## 1. Concept — The form is the oldest API, and Next.js kept it honest

Every web mutation before JavaScript was a **form POST**: a `multipart/form-data` body, a `303 See Other` redirect, the new page. Server Functions **re-use that protocol unchanged** (module 29's "behind the scenes, actions use POST"): the `<form action={serverFunction}>` *is* the native form POST — the `action` attribute is the **action ID** (the module-29 build-time swap: the function's *implementation* never ships; the *attribute* is the opaque reference the browser can `POST` to).

**The two modes of one form** (the module's core — *one markup, two behaviors*):

| | **JS enabled** (the common case) | **JS disabled** (the guarantee) |
|---|---|---|
| Submit event | React **intercepts** the submit (no native navigation) | The browser does the **native `POST`** |
| Transport | The action is *invoked* via the framework's dispatcher (the same POST — module 29; the *RSC payload* comes back) | The same POST — the **server responds with the `303` + the new page's HTML** |
| The mutation | Runs **inside a `startTransition`** (automatic — the official line: passed to `action`/`formAction` = *automatic* startTransition) | Runs (the action is *just* a POST handler — no React needed to *run* it) |
| The result | The **RSC payload** re-renders the tree (the module-29 single roundtrip: the fresh data in the response) | The **303 redirect** → the browser *GETs* the new page (the full document) |
| Pending UX | `useFormStatus` (module 32) — the button shows "Saving…" | *None* (the browser's own "waiting" state) — the *form works; the feedback is the browser's* |
| Errors | The action's returned state / thrown error → the module-31 round-trip (the field errors *in the re-render*) | The action's error → a redirect to the page *with the error in the URL* (`?error=…` — the module-31's *JS-off* error path — the `searchParams` the *server* can read) |

**The mental-model sentence:** *a Server Function form is a native form whose `action` happens to point at a Server Function — the framework upgrades the response (the RSC payload, the soft re-render) when JS is present, and degrades to the 303 (the hard navigation) when it isn't.* The **upgrade is the bonus; the native POST is the floor.** That floor is *the progressive enhancement* — and it's *why* the capstone's forms are `<form>` + action, *not* the SPA's `onClick={() => fetch('/api/…')}` (the *fetch-form* has **no floor** — JS disabled, the button does *nothing*; the module-16 self-API anti-pattern, the *form* version).

**The 303 (POST-redirect-GET) is the *correctness* property** (the module-09-04 redirect table, the form's row): after a *successful* mutation, the action `redirect()`s (the module-29 step 5) — the browser's **refresh** re-`GET`s the *result* page (it does **not** re-`POST` the form — the *double-submit* is impossible by *protocol*; the "Are you sure you want to resubmit?" prompt is *gone* from the mutation path). A form that *returns* the new page directly (no redirect) makes refresh *re-submit* (the module-29's `redirect`-is-not-an-error line, the *form's* consequence) — **the capstone rule: a successful form action ends in `redirect()`** (the module-29 template's step 5, *enforced* by the form's refresh semantics).

## 2. Mental Model — `action` vs `formAction` (the two attachment points)

| Attachment | What it is | The capstone's use |
|---|---|---|
| `<form action={fn}>` | The form's **default** action (the `submit` event / the form's *implicit* submit button) | The **Save** (the form's primary mutation: create/update) |
| `<button formAction={fn}>` (inside the form) | A **per-button** action — that button's click submits the form to *that* function (the *other* buttons use the form's `action`) | The **Delete** (a form with a `Save` action + a `Delete` `formAction` button — *two mutations, one form*, each with its *own* server path) |

**The `formAction`'s argument flow** (the subtle one, official): the button's `formAction` receives the **same `FormData`** as the form's `action` — *plus* any *static* args you pass to the `formAction` call — **and** the button's own `<input name=… value=…>` contributions (a `<button name="action" value="delete">` inside a *single* action is the *old* "branch on a hidden field" pattern; `formAction` makes the branching *explicit* — no `if (formData.get('action') === 'delete')` switch in one action).

**The args rule** (module 29's, the form context, precise): `fn(staticArg1, staticArg2, …, formData)` — the **statics first, `FormData` last**; *in a form context, everything after the first static arg is `FormData`* (the form *is* the body — you can't pass a *second* static *after* the form data; the build enforces it). The capstone's pattern: **the entity's ID is the static arg** (the `updateProduct(slug: string, formData: FormData)` — the `slug` from the *URL* (the server's *route* knows it — the module-07's params) — *not* from a hidden input *when the server can derive it* (the module-19's "derive, don't take" — the *hidden input* is for what the *server cannot derive*: a *selected option* (a `<select>` value — *that's in the FormData anyway* — the hidden input is for **static per-row context** in a *list of buttons* (the *per-row* `formAction`'s *static arg*: the row's ID the *server* can't derive from the form — the module-29's `markShipped(orderId)` as a *form button* in a table row)).

```tsx
// The per-row button: the formAction's static arg is the row's ID (the server can't
// derive it from the FormData — the row IS the context):
{orders.map((o) => (
  <tr key={o.id}>
    <td>{o.number}</td>
    <td><span>{o.status}</span></td>
    <td>
      <button formAction={markShipped.bind(null, o.id)}    // the static arg: this row's ID
              disabled={o.status === 'shipped'}>
        Mark shipped
      </button>
    </td>
  </tr>
))}
// markShipped(orderId: string) — the action takes the ID (the static), the FormData is
// absent (a button's formAction with a static arg and no form fields: the FormData is
// still passed (an empty FormData) — the action's signature: (orderId: string, _formData: FormData)
// — the *official* form: the statics first, the FormData last (always present in a form context).
```

## 3. Architecture — The capstone's form inventory (the mutation surfaces, collected)

`FILE: docs/form-inventory.md` (the artifact — the module-29 action inventory's *form* subset, with the *progressive* column):

```md
| Form (feature) | action (module 29) | formAction buttons | JS-off behavior (the 303 target) | JS-off error path (module 31) |
|---|---|---|---|---|
| Product create (/products/new) | createProduct | — (the Save only) | 303 → /products/{slug} (the fresh detail) | 303 → /products/new?error=validation (+ the field errors *in the re-render* when JS; the *summary* in the URL when off — module 31's split) |
| Product edit (/products/{slug}/edit) | updateProduct(slug, fd) | delistProduct (the per-form `formAction` button) | 303 → /products/{slug} (edit) / /products (delist) | ?error=validation / ?delisted=1 |
| Order actions (the rows, /orders) | — (no form action — the row buttons are `formAction`-only) | markShipped(id, fd) · cancelOrder(id, fd) | 303 → /orders (the list, fresh) | ?error=… (per-row, the row's ID in the URL — module 31's row-errors) |
| Org profile (/settings/profile) | updateProfile | — | 303 → /settings/profile | ?error=validation |
| Org settings (the plan/billing toggle) | updateOrgSettings | — | 303 → /settings | ?error=… |
| Member invite (/settings/members) | inviteMember | removeMember(id, fd) (the per-row) | 303 → /settings/members | ?error=… |
| Login (/login) | signIn (Phase 10) | — | 303 → /dashboard (the module-09-04's login redirect) | 303 → /login?error=credentials (the module-10's error, *not* a field-error — the *summary* only) |
| Password change (/settings/security) | changePassword (Phase 10) | — | 303 → /settings/security | ?error=… |
| Payout settings (Phase 15-adjacent) | updatePayouts | — | 303 → /settings/payouts | ?error=… |
| Capstone feedback (the (marketing) "suggest a feature") | submitFeedback | — | 303 → /thanks (the *public* form — the JS-off floor matters *most* for *public* forms (the SEO/crawler floor, module 14)) | ?error=… |
```

**The inventory's rule (the module's gate):** *every* mutation in the capstone is a **`<form>` with a Server Function `action`** — **zero** `onClick={() => fetch(…)}` forms (the module-16 self-API anti-pattern, the *form* version — a *fetch-form* is a *review finding*: it has *no floor* (the JS-off user is *locked out of the mutation*), it *duplicates* the service path (the `fetch('/api/…')` → the BFF → the service — the module-16's *deleted* layer), and it *loses* the RSC re-render (the *manual* client state update — the module-12's wire format *bypassed*). The **exception** (documented, not the default): a mutation that is *inherently client-driven* (a **drag-and-drop reorder** — the *payload* is the *client's* drag state, not a form — a *event-handler* action call (module 29's invocation #2) is *correct there* (the *form* can't express a drag) — the *rule*: **forms for form-shaped data; event-handler actions for client-state-shaped data** (the module-33 decision matrix, the row).

## 4. Production Code — The product form, complete (both modes, one component)

`FILE: src/features/catalog/components/product-form.tsx` (production pattern — the form is a **Server Component's child** (no `'use client'` on the *form* — the *islands* are the *inputs* that need state (the module-32's `useFormStatus` button, the module-13's `TextField`): the *form* is server-rendered (the *floor* requires it — a `'use client'` *form* is *fine* (the *floor* survives — the `action` prop works in a client form too (the module-29's invocation #1: "forms in Server *and* Client components") — the *capstone* keeps the form *server* (the *inputs* are the islands): the *fewer* client components in the *form's* subtree, the *stronger* the floor)

```tsx
// PRODUCT FORM — the create/edit in one (the mode is the `product` prop's presence):
import { useFormStatus } from 'react-dom'
import { redirect } from 'next/navigation'
import { z } from 'zod'
import { createProduct, updateProduct, delistProduct } from '../product-actions'
import { productInput } from '../product-schema'        // the shared Zod (module 04: the *one* schema, client+server)
import { TextField } from '@/components/ui/text-field'  // the module-13 design system (the [CLIENT] island: the label+input+error)
import { SubmitButton } from './submit-button'          // the useFormStatus button (module 32's preview; §4.2)
import { FormError } from '@/components/ui/form-error'  // the form-level error banner (module 31's)

export function ProductForm({ product }: { product?: ProductDto }) {
  const isEdit = Boolean(product)
  // the JS-off error path (module 31's split): the 303 target's ?error=… — the server reads it:
  // (the searchParams are read in the PAGE (the server component), passed in — §4.3)
  return (
    // The action: the create OR the update (the static arg: the slug — the server *derives* it
    // from the route (module 29's "derive, don't take" — the *page* passes it, the *hidden input* is NOT the source):
    <form action={isEdit ? updateProduct.bind(null, product!.slug) : createProduct} className="space-y-4">
      <fieldset className="space-y-4">
        <legend className="text-sm font-medium">Product</legend>
        <TextField
          name="name"
          label="Name"
          defaultValue={product?.name}
          required
          // the field errors: the JS-on path (module 31's useActionState state — the page passes them in):
          error={isEdit ? undefined : undefined /* module 31 wires the per-field errors here */}
        />
        <TextField
          name="price"
          label="Price (USD)"
          type="number"
          inputMode="decimal"
          step="0.01"
          min="0"
          defaultValue={product ? String(product.price) : undefined}
          required
        />
        <TextField
          name="sku"
          label="SKU"
          defaultValue={product?.sku}
        />
        <textarea name="description" rows={4} defaultValue={product?.description}
                  aria-label="Description" className="…module-13's textarea…" />
      </fieldset>

      {/* the form-level error (module 31's FormState.error — the page passes the state in): */}
      <FormError />   {/* the JS-off path's ?error= summary is rendered by the PAGE (below), not here */}

      <div className="flex items-center gap-3">
        <SubmitButton>{isEdit ? 'Save changes' : 'Create product'}</SubmitButton>
        {isEdit && (
          // the formAction: the SECOND mutation on this form (the module-29's formAction —
          // the per-button action; the static arg: the slug; the FormData: the form's (ignored
          // by the delist — it takes only the slug):
          <button
            formAction={delistProduct.bind(null, product!.slug)}
            className="text-sm text-danger"
            // the module-13's confirm: the native confirm() is JS-off-ineffective (the floor:
            // the delist is a *destructive* action — the JS-off floor has NO confirm — the
            // capstone's choice: the delist's *service* is idempotent (module 09-04) + the
            // audit row (module 17) + the *soft-delete* (the delist is a *status*, reversible —
            // the destructive-confirm is the *permanent* delete's (Phase 11's) job):
          >
            Delist product
          </button>
        )}
      </div>
    </form>
  )
}
```

### 4.1 The form's *page* (the server side: the args, the errors, the floor's wiring)

`FILE: src/app/(app)/products/new/page.tsx` (production pattern — [SERVER])

```tsx
import { ProductForm } from '@/features/catalog/components/product-form'

export default function NewProductPage() {
  return (
    <div className="max-w-xl space-y-6">
      <h1 className="text-2xl font-semibold">New product</h1>
      <ProductForm />
    </div>
  )
}
```

`FILE: src/app/(app)/products/[slug]/edit/page.tsx` (production pattern — [SERVER]; the edit page wires the *both* error paths)

```tsx
import { notFound } from 'next/navigation'
import { headers } from 'next/headers'
import { z } from 'zod'
import { auth } from '@/lib/auth'
import { getProductBySlugFresh } from '@/services/catalog'   // the FRESH read (the edit form shows the *current* truth — module 17's uncached core — NOT the cached DTO (stale-on-edit is the module-19 ticket))
import { ProductForm } from '@/features/catalog/components/product-form'
import { FormError } from '@/components/ui/form-error'

const paramsSchema = z.object({ slug: z.string().regex(/^[a-z0-9-]+$/) })

export default async function EditProductPage({ params }: { params: Promise<{ slug: string }> }) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) notFound()                     // the module-29's gate (the *page* — the action re-checks (module 29's "the layout's gate ≠ the action's gate"))
  const { slug } = await params
  if (!paramsSchema.safeParse({ slug }).success) notFound()    // the module-04's boundary validation

  const product = await getProductBySlugFresh(session.organizationId, slug)   // the tenancy (module 17 rule 1)
  if (!product) notFound()

  // the JS-off error path (module 31's split): the 303's ?error=… — the server *reads* it
  // (the searchParams — the module-10's URL-state rule: the error is *URL state* (shareable,
  // refreshable, the no-JS floor)):
  const searchParams = await /* the page's searchParams prop (module 07) */ Promise.resolve({})
  // (in the real file: `export default async function EditProductPage({ params, searchParams }:
  //   { params: Promise<…>; searchParams: Promise<Record<string, string>> })` — the module-07's
  //   async searchParams (the module-04's verified behavior) — the error is a *string code* (the
  //   module-19-03's "no internals in the URL" — the *code*, not the message):
  //   const sp = await searchParams; const errorCode = sp.error as 'validation' | undefined)

  return (
    <div className="max-w-xl space-y-6">
      <h1 className="text-2xl font-semibold">Edit {product.name}</h1>
      {/* the JS-off summary (the code → the user-facing text — the mapping is the module-31's): */}
      {/* <FormError code={errorCode} /> */}
      <ProductForm product={product} />
    </div>
  )
}
```

### 4.2 The `SubmitButton` (the pending island — module 32's preview, the form's)

`FILE: src/features/catalog/components/submit-button.tsx` (production pattern — **[CLIENT]** (the `useFormStatus` is a hook — the *island*; the *form* around it is server — the module-13's "the smallest island" rule))

```tsx
'use client'

import { useFormStatus } from 'react-dom'   // the module-29's "automatic startTransition" — the *status* of *that* transition:

export function SubmitButton({ children }: { children: React.ReactNode }) {
  const { pending } = useFormStatus()        // the *pending* state of the *enclosing form's* action (the module-32's deep-dive: the pending gate, the label, the rollback)
  return (
    <button
      type="submit"
      disabled={pending}                     // the module-29's pending gate (the double-submit control — the *form's* version: the *native* submit is *disabled* while the action is pending)
      className="rounded-md bg-primary px-4 py-2 text-sm font-medium text-white disabled:opacity-60"
    >
      {pending ? 'Saving…' : children}
    </button>
  )
}
```

**The `useFormStatus`'s *floor* check** (the module's subtlety): the `SubmitButton` is a **client island** — but the *floor* (JS-off) doesn't *use* it (the JS-off submit is the *native* one — the `disabled={pending}` is *never evaluated* — the *button is enabled* (the HTML's default) — the native POST proceeds (the floor *works*) — the `useFormStatus` *enhances* (the pending label) — it *never gates* (a `disabled` that's *JS-on-only* is the *correct* enhancement — the *floor* is the *HTML*, the *island* is the *polish*). **The rule (the module's standing line): *no client island may gate a floor-critical behavior* — the `disabled` in an island is *enhancement* (the double-click polish); the *correctness* gate is the *server's* (the service's idempotency, module 09-04) — the module-29's double-click experiment, the *form's* version: the *floor's* double-submit (the JS-off *Enter key mash*) is *safe* because the *service is idempotent* (the module-09-04's status-transition guard) — *not* because of the `disabled` (the JS-off has no `disabled` to respect).**

### 4.3 The `hidden` input, precisely (when the floor needs it)

The *static arg* (the `slug`) is *derived* (the route — the module-29's "derive, don't take") — **the hidden input is for the *formAction* per-row statics the server *can't* derive** (the §2's `markShipped.bind(null, o.id)` — the *row's* ID — the *form* is the *list's* form (the *one* form, the *many* row buttons — the *server* can't derive *which row* from the FormData — the `bind` passes it as the *static arg* (the *module-29's* additional-args — the *formAction's* version): **`bind` is the formAction's "hidden input"** (the *server-side* hidden input — the *value* is in the *action reference's* serialized args (the *module-14's regime 2*: the *static args* are *serialized into the POST body* (the *module-14's* "plain-serializable args" — the *ID is a string* (serializable) — the *hidden `<input type="hidden">` is the *FormData* version (the *value* is in the *multipart body*) — *both* work; the *capstone* uses `bind` for the *per-row* (the *cleaner* — no *name-collision* (a *hidden input*'s *name* is in the *FormData* (a *form with 50 rows* = *50 hidden inputs* named `orderId` — the *last one wins* (the *FormData* collision) — the `bind`'s *per-button* static *avoids* the collision) — the *hidden input* is for the *single* per-form static (the *CSRF token* — module 29's §6 — *one* token, *one* input, *no collision*).

## 5. Common Mistakes (the form failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The fetch-form** (`<button onClick={() => fetch('/api/products', {method:'POST', body:…})}>` + the *manual* state update) | The JS-off user is *locked out* (the button does *nothing*); the BFF's *duplicated* path (module 16); the *manual* re-render (the module-12's wire format *bypassed*; the *stale* client state — the *optimistic* that *never reconciles* (the module-32's rollback *without* the server's truth)) | The `<form action={…}>` (the *floor* + the *RSC re-render* + the *one* service path (the module-16's deletion) — the *inventory's* rule: *zero* fetch-forms (the *review finding*) |
| **The `action` on a *client* form with a *client-defined* "action"** (`const action = async (fd) => { … }` in a `'use client'` file, passed to `<form action={action}>`) | Build error (*the function isn't a Server Function* — no `'use server'`) — the *confusion*: a *client* async function is *not* an action (the *module-29's* "cannot define in a client component") — the *form* needs a *Server Function reference* (imported from the `'use server'` file) | The action is *imported* from `features/*/actions.ts` (the module-29's file-level) — the *client form* *calls* it (the *module-29's invocation #1: "forms in Server *and* Client components" — the *form* is client, the *action* is server — the *boundary* is the *import*) |
| **The refresh re-submit** (the action *returns* the new page *directly* — no `redirect()`) | The *browser's* "resubmit?" prompt (the *module-29's* redirect-is-not-an-error, the *form's* consequence) — the *double-mutation* (the *refresh* re-POSTs — the *createProduct* *twice* (the *duplicate product* — the *module-09-04's* idempotency *doesn't cover* the *create* (a *create* is *not idempotent* (the *two products*) — the *redirect is the *create's* *correctness* control)) | The *successful form action ends in `redirect()`* (the module-29 step 5, *enforced* — the *inventory's* "303 target" column, *per form*) |
| **The `disabled` that gates the floor** (a *client* island button that's `disabled` *by default* until JS hydrates (`disabled={!mounted}` — the module-13's mounted-gate *misused* as a *gate*)) | The JS-off user sees a *disabled* button (the *floor is broken* — the *mutation is *locked out*) — the *mounted-gate* is for the *hydration mismatch* (the module-13's "render both, swap on mount" — the *polish*, *not* the *gate*) | The *island's* `disabled` is *pending-only* (the `useFormStatus`'s `pending` — the *enhancement*) — the *correctness* is the *server's* (the *idempotency*) — the *module-4.2's* rule: *no island gates a floor-critical behavior* |
| **The hidden-input collision** (the *per-row* hidden `<input name="id" value={row.id}>` ×50 rows, *one* action that reads `formData.get('id')`) | The *last* row's ID (the *FormData* collision — the *wrong row* mutated) | The `formAction.bind(null, row.id)` (the *per-button* static — the *§4.3's* rule: `bind` for the *per-row*, the *hidden input* for the *single per-form* (the *CSRF token*)) |
| **The *validation* in the *form component*** (the Zod `parse` in the *component* that *renders* the form) | The *component* runs *on the server render* (the *page's* render — the *formData is *absent* (the *render* has *no* form data (the *form* is *rendered* before it's *submitted*) — the *parse* *crashes* (the *undefined* formData) or *validates nothing* (the *empty* parse) | The *validation is in the *action* (the module-29 step 2 — the *formData is *the action's* input — the *module-31's* home — the *form component* *renders* the *fields* + the *errors* (the *module-31's* state) — the *parse* is *never* in the *render path* |
| **The *searchParams error* as a *message*** (`?error=The price must be a number between 0 and 9999`) | The *module-19-03's* "no internals in the URL" *violation* (the *message* is *shareable* (the *URL is *copied*) — the *error's *specifics* (the *price bound* — the *validation logic*) *leaked* (the *attacker's* *fingerprint*) + the *URL's *length* (the *share* is *ugly*) | The *URL carries the *code* (`?error=validation` — the *module-4.1's* line — the *mapping* (code → *user-facing text*) is the *module-31's* (the *server* renders the *text* — the *URL is *the code only*) |

## 6. Security Notes (the form's *specific* surface)

- **The CSRF token (the module-29's §6, the *form's* first line)**: the *capstone's* posture (module 29: `sameSite=lax` + the `allowedOrigins` + the *in-function checks*) — the *form adds* the *token* (the *module-30's* *extra* line, for the *top-level form POST* the `lax` *doesn't* block (the module-29's precise line)): a *hidden input* with a *session-bound token* (the *module-10's* session — a *random* value, *server-generated*, *per session*, *not the session ID itself* (the *module-10-03's* "don't put the ID in the cookie's sibling" — the *token is *derived* (a *signed* value — the *module-10's* *signing key* — the *action verifies* (the *token matches the session* (the *module-29's* step 1.5: the *token check* is *before* the *validation* (the *fastest* rejection — the *forged* form *fails* before the *Zod* (the *module-19's* "reject early, reject cheap"))). **The *floor's* token**: the *hidden input* is *server-rendered* (the *form is *server* (the *module-4's* rule) — the *token is *in the *HTML* (the *JS-off* *submit* *includes* it (the *floor is *CSRF-safe* (the *JS-on* is *the same token* (the *one token, *both modes*) — the *token is *not* a *JS feature* (the *module-30's* standing line: **the CSRF protection is *in the HTML* (the *hidden input*) — *never* in a *client island* (a *token in a *`useState`* is *absent* in the *JS-off* *submit* (the *floor's* *CSRF hole*) — the *module-4.2's* rule, *applied to the token*: *no island owns a floor-critical security value*.
- **The `FormData` is *multipart* (the *file upload* is the *module-16's* *subject*)**: the *form's* `<input type="file" name="image">` is *in the *multipart body* (the *action receives* it in the *formData* (`formData.get('image')` → a *`File`* (the *module-16's* *verified behavior*) — the *module-30's* form is the *module-16's* *vehicle* (the *upload is *a form field* (the *floor: the *file upload works *JS-off* (the *native* multipart POST) — the *module-16's* *progressive* line (the *drag-drop is *the enhancement* (the *module-33's* row: *client-state-shaped* → the *event-handler* action (the *module-29's* invocation #2) — the *fallback is *the form field*).
- **The *action's* *args* are *client-supplied* (the module-29's §6, the *form's* *args**): the *static arg* (the `slug`) is *derived* (the *route* — the *module-29's* "derive, don't take") — *but* the *derived value is *still validated* (the *module-4.1's* `paramsSchema` — the *route param is *URL-shaped* (the *attacker's* `slug` — the *module-04's* boundary) — the *derive* is the *source* (the *not the hidden input*) — the *validate is *the *boundary* (the *module-04's* *Zod* — the *slug's* *regex* (the *module-24's* *slug schema*)).
- **The *error path* is *the attack surface* (the module-31's *round-trip*)**: the *field errors* are *user-facing text* (the *module-19-03's* "no internals") — the *Zod's* *message* is *the default* ("Invalid input" — the *module-04's* *Zod vs TS* — the *default message is *the *safe* one (the *custom message is *the *design* (the *module-13's* *error copy*) — the *module-31's* *rule: *the *field error is *a string the *user* *reads* (the *not* the *Zod's* *path* (`price[0]`) — the *not* the *schema's* *shape* (the *leak*) — the *module-31's* *mapping* (the *Zod *issue* → the *field's* *label* (the *module-13's* *design system* owns the *error *copy*)).

## 7. Performance Notes

- **The form's *TTFB* is the *page's* (the *form's* *render*)**: the *form is *server-rendered* (the *module-24's* *static/dynamic* — the *edit form* is *dynamic* (the *session* + the *product read* (the *module-25's* *request-scoped* rows) — the *new form* is *static* (the *no session* — the *(marketing)*'s *feedback form* is *prerendered* (the *module-24's* *SHAPE 1* — the *form's* *shell* is *the CDN* (the *floor is *fast* (the *HTML is *the edge*) — the *form's* *performance* is the *page's* *performance* (the *module-24's* *decision table*, the *form's* *row*).
- **The *submit's* latency is the *action's* (the module-29's *20–100ms*) + the *redirect's* *page* (the *303's* *GET* — the *module-09-04's* *redirect* — the *two hops* (the *POST* + the *GET* (the *module-29's* *single roundtrip* is the *JS-on* (the *RSC payload* is the *POST's* *response* (the *one hop*) — the *JS-off* is *two hops* (the *303* + the *GET*) — the *floor's* *cost* is the *extra hop* (the *honest* *number: the *JS-off* *submit is *slower* (the *303's* *GET is *a full page* (the *not* the *RSC payload*) — the *floor is *correctness* (the *the *speed is *the JS-on's* *privilege* (the *module-30's* *standing line: **the floor is *slower* (the *303's* *full page*) — the *JS-on is *faster* (the *RSC payload's* *re-render*) — the *both work* (the *the *no choice* (the *the *the progressive *enhancement is *the speed* (the *the *the floor is *the *right*).
- **The *pending UX* is the *perceived* (the module-29's §7, the *form's*)**: the *`useFormStatus`'s* *label* (the *"Saving…"*) is the *perceived speed* (the *real* latency is the *action's* (the *20–100ms*) — the *label is *the *polish* (the *module-32's* *deep-dive*) — the *form's* *pending is *the *module-32's* *subject* (the *module-30's* *preview is *the *`SubmitButton`* (the *§4.2*) — the *module-32's* *optimistic* is the *form's* *optional* enhancement (the *create's* *optimistic is *risky* (the *module-29's* *create is *not idempotent* (the *optimistic "created" is *a *lie* if the *server* *rejects* (the *module-32's* *rollback*) — the *edit's* *optimistic is *safe* (the *optimistic "saved" is *the *previous value's* *restore* (the *module-32's* *standard*) — the *module-32's* *decision* (the *per-mutation* *optimistic* (the *the *the create is *no* (the *the *the edit is *yes* (the *the *the delete is *no* (the *the *the module-32's* *table*).

## 8. Exercise

**Beginner.** Build the *product form* (the §4) — the *create* (`/products/new`) — *with the stub* `createProduct` (the module-29's *five-step* template, the *create* variant: the *session* → the *Zod* (the *`productInput`* — the *module-31's* *schema* (the *§4's* *import*)) → the *service* (the *stub* — the *`delay(300)`* + the *fake* *slug*) → the *`revalidateTag`* (the *`'products'`*) → the *`redirect('/products/' + slug)`*). **The floor test**: the *DevTools* → the *Settings* → the *"Disable JavaScript"* (the *module-28's* *protocol*, the *form's* version) — *submit the form* (the *native POST*) — the *303* → the *fresh product page* (the *floor *works*) — *re-enable JS* — the *soft re-render* (the *module-29's* *single roundtrip*) — *screenshot both* (the *floor's* *303* in the *Network tab* (the *303* → the *200 GET*) + the *JS-on's* *RSC payload* (the *`RSC: 1` header* — the *module-12's* wire format) — the *two modes, one form, *evidenced*).

**Intermediate.** The *`formAction`* lab (the §2's *per-row* buttons): the *orders table* (the *module-25's* `OrdersTable`) — *add* the *per-row* `markShipped` button (the *`formAction={markShipped.bind(null, o.id)}`* — the *module-30's* *§2's* code) — *the form is *the *table's* form* (the *one* form, the *many* row buttons) — *verify*: (a) the *JS-on*: the *click* → the *action runs* (the *row's* ID is the *arg* (the *DevTools* console: the *POST body* — the *static arg* is *there* (the *module-14's* regime 2 — the *serialized arg*) (b) the *JS-off*: the *click* → the *native POST* → the *303* → the *list* (the *fresh* — the *row is *shipped*) — (c) the *collision check*: the *50 rows*, the *`bind`* — the *no* *hidden-input collision* (the *§4.3's* rule — the *`bind`'s* *per-button* static — the *FormData is *empty* (the *no* *`id` field* (the *collision is *impossible*)).

**Production.** The *form-inventory audit* (the module-33's *decision matrix's* *form rows*): for *every* form in the capstone (the *inventory's* *10 rows*) — *verify*: (a) the *floor* (the *JS-off* *submit works* (the *DevTools disable* — the *per-form* (b) the *303 target* (the *module-29's* step 5 — the *redirect is *present* (the *the *no re-submit* (the *refresh test* (c) the *CSRF token* (the *hidden input* is *server-rendered* (the *grep: the *`<input type="hidden" name="_csrf"`* is *in the *form* (the *not* in an *island*) (d) the *static arg* (the *derived* (the *route*) — the *not* the *hidden input* (the *except* the *CSRF token* (e) the *no fetch-form* (the *grep: the *`fetch(.*POST`* in a *`'use client'` form component* — the *zero* (the *module-16's* anti-pattern). The *five checks per form* (the *50 cells*) are the *form's security/correctness matrix* — the *module-20's* *test phase's* *starting point* (the *Playwright's* *form suite* is *generated from this matrix* (the *module-20's* *e2e* — the *floor's* *test is the *JS-off* (the *Playwright's* *`page.evaluate(() => { … })`* to *disable JS* (the *module-20's* *technique* — the *e2e's* *floor test*).

## 9. Architecture Challenge

**Prompt:** The *checkout's* *address form* (the *module-12's* *checkout*, the *form's* *version*): the *user* fills in *name, line1, line2, city, state, zip, country* — the *submit* → the *order* is *created* (the *module-25's* *order*, the *address is *a *field*) — *then* the *shipping estimate* is *needed* (the *carrier's API* — the *module-18's* *external* — the *address* is the *input*) — *the* *UX requirement*: the *estimate* updates *as the user types* (the *live* — the *module-27's* *streaming* — the *estimate is *a *hole* that *re-renders* on *address change*) — *but* the *floor* (the *JS-off*) must *still* *complete the checkout* (the *order is *created, the *estimate is *absent* (the *the *the price is *the *server's* *final* (the *module-12's* *"the price is *recomputed* at *submit"* (the *the *the estimate is *a *preview* (the *the *the truth is *the *submit's* *recompute*).

**Design**: the *form* (the *floor: the *native POST* → the *action* → the *order* → the *303* → the *order confirmation* (the *estimate is *not* in the *floor* (the *the *the estimate is *the *JS-on* *enhancement*) — the *JS-on*: the *live* *estimate* (the *what re-renders* (the *module-27's* *boundary* — the *estimate is *a *section* that *awaits* the *carrier API* (the *module-18's* *parallel* — the *address is *the *arg*) — the *address is *URL state* (the *module-10's* *URL-state* rule — the *estimate's* *arg* is the *address* (the *the *the address is *in the *URL* (the *`?line1=…&city=…`*) — the *module-10's* *four-owner* table: the *address is *the *URL's* (the *shareable* (the *the *the estimate is *recomputed* on the *URL change* (the *soft navigation* (the *the *the form's *fields* are *synced* to the *URL* (the *module-10's* *commit* pattern — the *debounced* (the *module-10's* *commit* (the *the *the "as the user types" is *the *debounced URL commit* (the *the *the module-10's* *debounce* (the *300ms*) — the *estimate* *re-renders* (the *module-27's* *hole* (the *the *the streaming*), and *when the user submits* (the *form* → the *action* → the *order* (the *the *the estimate is *the *last URL state* (the *the *the order's *address is *from the *formData* (the *the *the server *recomputes* the *price* (the *module-12's* *line*) — the *estimate's* *value* is *not* the *order's* *price* (the *the *the two are *separate* (the *the *the estimate is *a *preview* (the *the *the order is *the *truth*).

Produce: the *form* (the *floor's* *fields* (the *no JS* (c) the *CSRF token* (d) the *submit's* *action* (the *module-29's* *five steps* — the *address is *in the *formData* (the *the *the order's *address is *the *formData's* (the *not* the *URL's* (the *the *the URL is *the *estimate's* *arg* (the *the *the formData is *the *order's* *input*) (e) the *303 target* (the *order confirmation*) — the *JS-on*: the *URL-sync* (the *module-10's* *commit* (the *debounce* (f) the *estimate* *section* (the *module-27's* *boundary* — the *`Suspense`* + the *carrier API* (the *module-18's* *external* — the *rate limit* (the *module-18's* *the* *carrier's* *cost* (the *the *the debounce is *the *rate-limit* control (the *module-18's* *line*) — the *skeleton* (the *module-27's* *§4.3*) — and the *why* (the *floor is *the *order* (the *the *the estimate is *the *enhancement* (the *the *the two are *separate* (the *the *the no *single* *code path* (the *the *the module-12's* *"the price is *recomputed* at *submit"* (the *the *the estimate is *a *preview* (the *the *the order is *the *truth*) (the *the *the module-30's* *standing line: **the floor is *the *correctness* (the *the *the enhancement is *the *perception* (the *the *the no *enhancement may *change the *floor's* *truth* (the *the *the estimate is *never* the *order's* *price* (the *the *the submit *recomputes*).

<details>
<summary>Model answer</summary>
**The *form* (the *floor*):**

```tsx
// src/features/checkout/components/address-form.tsx (the *floor* — the *no-JS* *checkout*):
import { z } from 'zod'
import { placeOrder } from '../checkout-actions'   // the *module-29's* *five steps* (the *address is *in the *formData*)
import { SubmitButton } from '@/components/ui/submit-button'   // the *module-30's* §4.2 (the *pending island*)
import { FormError } from '@/components/ui/form-error'

export function AddressForm({ csrfToken }: { csrfToken: string }) {
  return (
    <form action={placeOrder} className="space-y-4">
      {/* the *CSRF token* (the *module-30's* §6 — the *server-rendered* *hidden input* (the *floor is *CSRF-safe*) (the *the *the token is *derived from the *session* (the *module-10's* *signing*) — the *page* renders it (the *not* an *island*) (the *module-4.2's* rule: *no island owns a floor-critical security value*): */}
      <input type="hidden" name="_csrf" value={csrfToken} />
      <fieldset className="grid gap-4 sm:grid-cols-2">
        <label className="block text-sm"><span>Name</span>
          <input name="name" required className="…" /></label>
        <label className="block text-sm"><span>Line 1</span>
          <input name="line1" required className="…" /></label>
        <label className="block text-sm"><span>Line 2 (optional)</span>
          <input name="line2" className="…" /></label>
        <label className="block text-sm"><span>City</span>
          <input name="city" required className="…" /></label>
        <label className="block text-sm"><span>State</span>
          <input name="state" required className="…" /></label>
        <label className="block text-sm"><span>ZIP</span>
          <input name="zip" required inputMode="numeric" className="…" /></label>
        <label className="block text-sm sm:col-span-2"><span>Country</span>
          <select name="country" required defaultValue="US" className="…">
            <option value="US">United States</option>
            <option value="CA">Canada</option>
            {/* …the *supported* countries (the *module-24's* *i18n-adjacent* — the *country is *a *code* (the *not* a *name* (the *module-14's* *i18n* — the *code is *the *stable* (the *name is *the *translation*) */}
          </select></label>
      </fieldset>
      <FormError />   {/* the *module-31's* *form-level* error (the *JS-on* *round-trip* — the *JS-off* is the *page's* *`?error=`* (the *§4.1's* *pattern*) */}
      <div className="flex items-center gap-3">
        <SubmitButton>Place order</SubmitButton>
        {/* the *estimate* is *NOT* in the *form* (the *it's a *sibling section* (the *module-27's* *boundary* — the *form's* *submit is *independent* of the *estimate* (the *the *the estimate is *a *preview* (the *the *the form is *the *truth*) */}
      </div>
    </form>
  )
}
```

**The *page* (the *wiring*):**

```tsx
// src/app/(app)/checkout/page.tsx (the *server* — the *estimate's* *arg* is the *URL* (the *module-10's* *URL-state*):
import { headers } from 'next/headers'
import { Suspense } from 'react'
import { auth } from '@/lib/auth'
import { getCart } from '@/services/cart'             // the *module-25's* *cart* (the *request-scoped* (the *module-25's* *NOT-CACHED* row — the *cart is *the *session's* (the *the *the cart is *never cached* (the *module-20's* *auth-relevant*) */
import { AddressForm } from '@/features/checkout/components/address-form'
import { ShippingEstimate } from '@/features/checkout/components/shipping-estimate'   // the *module-27's* *hole* (the *below*)
import { EstimateSkeleton } from '@/components/estimate-skeleton'

export default async function CheckoutPage({ searchParams }: { searchParams: Promise<Record<string, string>> }) {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) redirect('/login')    // the *module-29's* gate (the *page*)
  const cart = await getCart(session.user.id)          // the *request-scoped* (the *module-25's* row)
  if (cart.items.length === 0) redirect('/dashboard')  // the *empty cart* (the *module-24's* *SHAPE 4* — the *checkout is *fully request-scoped*)

  const sp = await searchParams                          // the *module-07's* async searchParams (the *module-04's verified*)
  // the *address* is *URL state* (the *module-10's* *four-owner* table: the *address is *the *URL's* (the *shareable* (the *the *the estimate is *recomputed* on the *URL change*) (the *the *the URL is *the *estimate's* *arg* (the *the *the form is *the *order's* *input* (the *the *the two are *separate* (the *module-30's* line):
  const address = {
    line1: sp.line1 ?? '', line2: sp.line2 ?? '', city: sp.city ?? '',
    state: sp.state ?? '', zip: sp.zip ?? '', country: sp.country ?? 'US',
  }

  return (
    <div className="grid gap-6 lg:grid-cols-2">
      <div className="space-y-6">
        <CartSummary items={cart.items} />   {/* the *module-25's* *cart* (the *request-scoped*) */}
        <h2 className="text-lg font-medium">Shipping address</h2>
        <AddressForm csrfToken={await getCsrftoken(session.user.id)} />   {/* the *module-30's* §6 — the *server-rendered* token */}
      </div>
      {/* the *estimate* is *a sibling section* (the *module-27's* boundary — the *form's* submit is *independent*): */}
      <Suspense fallback={<EstimateSkeleton />}>
        <ShippingEstimate address={address} />
      </Suspense>
    </div>
  )
}

// the *estimate* (the *module-27's* *hole* — the *carrier API* (the *module-18's* *external*):
async function ShippingEstimate({ address }: { address: AddressDto }) {
  // the *carrier API* is *external* (the *module-18's* *parallel topology* — the *no* *caching* (the *the *the estimate is *a *preview* (the *the *the no *cache* (the *module-22's* *profile* — the *estimate is *request-scoped* (the *the *the no *tag* (the *the *the no *invalidation* (the *the *the estimate is *a *preview* (the *module-20's* *five questions*: the *WHO invalidates* is *no one* (the *the *the it's *a *preview* (the *module-25's* *NOT-CACHED* row, the *estimate's* version):
  const rate = await getShippingRate(address)   // the *module-18's* *external* (the *rate limit* (the *module-18's* line — the *debounce is *the control* (the *below*))
  return <EstimateCard rate={rate} />
}
```

**The *JS-on* *URL-sync* (the *module-10's* *commit* — the *island*):**

```tsx
// src/features/checkout/components/address-sync.tsx (the *island* — the *form fields' *URL commit* (the *module-10's* *commit pattern* — the *debounce* (the *300ms* — the *module-10's* *debounce*):
'use client'
import { useDeferredValue, useEffect, useRef } from 'react'
import { useRouter, useSearchParams } from 'next/navigation'

export function AddressSync() {
  const router = useRouter()
  const searchParams = useSearchParams()
  const debounced = useRef<ReturnType<typeof setTimeout> | null>(null)

  // the *form's* *fields* are *synced* to the *URL* (the *module-10's* *commit* — the *debounced* (the *the *the "as the user types" is *the *debounced URL commit* (the *module-10's* line):
  useEffect(() => {
    const form = document.querySelector<HTMLFormElement>('form[action]')   // the *form* (the *server-rendered* — the *island* *reads* it (the *module-13's* *the* *smallest island* — the *the *the form is *not* the *island's* (the *the *the island is *a *sibling* that *observes* the *form* (the *module-13's* *the* *no island owns a floor-critical element* (the *module-4.2's* rule, the *sync's* version: the *island* *reads* the *form* (the *not* *wraps* it (the *the *the form is *the floor's* (the *the *the island is *the enhancement's*)
    if (!form) return
    const onInput = () => {
      if (debounced.current) clearTimeout(debounced.current)
      debounced.current = setTimeout(() => {
        const fd = new FormData(form)
        const next = new URLSearchParams()
        for (const key of ['line1', 'line2', 'city', 'state', 'zip', 'country']) {
          const v = fd.get(key)
          if (typeof v === 'string' && v) next.set(key, v)
        }
        router.replace(`?${next.toString()}`, { scroll: false })   // the *module-10's* *commit* (the *`replace`* — the *no history spam* (the *module-10's* *line*) — the *soft navigation* (the *the *the estimate *re-renders* (the *module-27's* *hole* (the *the *the streaming*))
      }, 300)   // the *debounce* (the *module-10's* — the *rate-limit* control (the *module-18's* line: the *carrier's API is *costly* (the *the *the debounce is *the *throttle*)
    }
    form.addEventListener('input', onInput)
    return () => { form.removeEventListener('input', onInput); if (debounced.current) clearTimeout(debounced.current) }
  }, [router])

  return null   // the *island* is *a *behavior* (the *the *the no *DOM* (the *module-13's* *the* *headless island*)
}
```

**The *why* (the *three* *separations*, the *module's* *standing line*):**
1. **The *floor is *the *order***: the *native POST* → the *`placeOrder`* (the *module-29's* *five steps* — the *address is *in the *formData* (the *the *the order's *address is *the *formData's* (the *not* the *URL's* (the *the *the URL is *the *estimate's* *arg*) — the *303* → the *order confirmation* (the *the *the estimate is *absent* (the *the *the price is *the *server's* *final* (the *module-12's* line: *the price is *recomputed* at *submit* (the *the *the estimate is *a *preview* (the *the *the order is *the *truth*) — the *floor's* *checkout is *complete* (the *the *the no JS is *a *blocked checkout* (the *module-30's* *the* *floor is *the *correctness*).
2. **The *estimate is *the *enhancement***: the *URL-sync* (the *module-10's* *commit* — the *debounce* (the *300ms*) — the *estimate's* *section* (the *module-27's* *hole* — the *carrier API* (the *module-18's* *external* — the *rate limit* (the *module-18's* line — the *debounce is *the control*) — the *skeleton* (the *module-27's* §4.3) — the *streaming* (the *module-27's* *hole* — the *estimate lands* when the *carrier responds* (the *the *the no *form* *dependency* (the *the *the form's *submit is *independent* of the *estimate* (the *module-27's* *the* *no boundary *spans* the *form* (the *the *the estimate is *a *sibling* (the *module-27's* *rule 1* — the *boundary is *around the *query* (the *the *the query is *the *carrier* (the *the *the form is *not* the *query*).
3. **The *two are *separate***: the *address is *in the *URL* (the *estimate's* *arg* (the *the *the shareable* (the *module-10's* *URL-state* — the *the *the user can *share* the *estimate's* *URL* (the *the *the estimate is *reproducible*) — the *address is *in the *formData* (the *order's* *input* (the *the *the order's *address is *the *submitted* *value* (the *the *the no *sync* *dependency* (the *the *the form *doesn't* *read* the *URL* (the *the *the URL *doesn't* *write* the *order* (the *the *the two are *independent* (the *module-30's* *the* *no single code path* (the *the *the module-12's* *"the price is *recomputed* at *submit"* (the *the *the estimate is *a *preview* (the *the *the order is *the *truth*) (the *the *the module-30's* *standing line: **the floor is *the *correctness* (the *the *the enhancement is *the *perception* (the *the *the no *enhancement may *change the *floor's* *truth* (the *the *the estimate is *never* the *order's* *price* (the *the *the submit *recomputes* (the *module-12's* *line, the *checkout's* *enforcement*)).
**The *generalization* (the *form + *enhancement* *pattern*, the *module's* *standing rule*): **a *form with a *live* *companion* (the *estimate, the *preview, the *suggestion*) is *the *form* (the *floor: the *native POST* → the *action* → the *truth* → the *303*) + *a sibling section* (the *enhancement: the *URL-sync* (the *module-10's* *commit* — the *debounce* (the *rate-limit*) + the *hole* (the *module-27's* *boundary* — the *external* (the *module-18's*) — the *skeleton* (the *module-27's* §4.3)) — the *two are *separate* (the *the *the form is *the *truth* (the *the *the companion is *the *preview* (the *the *the no *single* *code path* (the *the *the companion's *value is *never the *form's* *submitted value* (the *the *the submit *recomputes* (the *module-12's* *line*) (the *module-30's* *the* *no enhancement may *change the *floor's* *truth*).
</details>

## 10. Official Documentation

- Forms (the `action` prop, the `formAction`, the additional arguments, the progressive enhancement): https://nextjs.org/docs/app/guides/forms
- Mutating Data (the POST-only, the startTransition's automatic wrap): https://nextjs.org/docs/app/getting-started/mutating-data
- `useFormStatus` (the pending status — module 32's deep-dive): https://react.dev/reference/react-dom/useFormStatus
- `redirect` (the 303, the POST-redirect-GET): https://nextjs.org/docs/app/api-reference/functions/redirect
- Server Actions and Mutations (the security, the build-time mechanics): https://nextjs.org/docs/app/guides/server-actions
- Handling Errors (the error round-trip — module 31's): https://nextjs.org/docs/app/guides/handling-errors

## 11. What You Should Know Before Continuing

- [ ] I can state the *two modes of one form* (the JS-on: the *soft* re-render; the JS-off: the *303* hard navigation) — the *floor is the *native POST* (the *the *the upgrade is the *RSC payload*)
- [ ] I know the *`action` vs `formAction`* (the *form's* default; the *per-button*) — and the *args rule* (the *statics first, the *FormData last*; the *`bind` for the *per-row* statics (the *no hidden-input collision*)
- [ ] I can write the *product form* (the §4) — the *floor's fields*, the *CSRF token* (the *server-rendered* hidden input), the *`SubmitButton`* (the *`useFormStatus` island — the *enhancement, not the gate*)
- [ ] I know the *303's correctness property* (the *refresh doesn't re-submit* — the *successful form action ends in `redirect()`*)
- [ ] I know the *no-fetch-form* rule (the *inventory's* gate: *zero* `fetch`-forms — the *floor + the *RSC re-render + the *one service path*)
- [ ] I know the *no-island-gates-the-floor* rule (the *`disabled` is the enhancement*; the *correctness is the server's* (the *idempotency*) — the *CSRF token is never in an island*)
- [ ] I've done the *floor test* (the *DevTools' disable JS* — the *303* — the *evidenced*) and the *form-inventory audit* (the *five checks per form*)

**Next:** Module 31 — Validation & Error Round-Trips (the *Zod in the action*; the *field + form errors returned to the form* — the *`useActionState`* (the *React 19's* hook) — the *JS-on* *round-trip* (the *field errors in the re-render*) + the *JS-off* *path* (the *`?error=` code in the URL*) — the *two paths, one validation*).
