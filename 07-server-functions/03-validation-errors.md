# Module 31 — Validation & Error Round-Trips: Zod in Actions, Errors Back to the Form

**Phase 7: Server Functions / Actions · Module 31 of 101**

> **Where does this run?** Validation is **`[SERVER]`** (the action's Zod parse — the module-29 step 2; the *source of truth*). The *errors* travel **`[BOTH — BOUNDARY]`** (down: the `FormState` DTO, serialization regime 1, module 14; up: the `FormData`, regime 2). The round-trip's *state* lives in the form's **`[CLIENT]`** island (`useActionState`) *and* — for the JS-off floor — in the **URL** (the `?error=` code, module 30's floor path). **Two paths, one validation.**

---

## 1. Concept — The round-trip, precisely

A form submit that *fails validation* must get back to the user: **the same form, their values preserved, the errors attached to the right fields.** The pre-19 way (the "searchParams pattern" — the error in the URL after a redirect) *works for JS-off but loses the field values* (the 303 re-renders an *empty* form — the user re-types) and *leaks error details in the URL* (module 30's `?error=` is a *code*, not the field errors). The **React 19 answer** is **`useActionState`** (the verified current API — React 19.2, module 01's stack):

```tsx
const [state, formAction, isPending] = useActionState(action, initialState)
// <form action={formAction}>  — formAction wraps the Server Function + carries the state
```

**The mechanism** (the *official* React model — `useActionState` is a *React* hook (it's what makes the `action` prop work with *returned state* — the Next.js `forms` guide's "Server Functions return a value" + the hook, the *one* modern pattern):

1. The form submits (the JS-on path: the hook *intercepts*, like the module-29's startTransition — the *automatic* wrap, module 30).
2. The action runs **with two arguments**: `(prevState, formData)` — the *previous state* (the last errors, if any — the *re-validation* on a *re-submit* sees what was wrong before) + the *FormData* (the *new* input).
3. The action **returns** a new state (a **`FormState`** — module 31's DTO: `{ error?: string; fieldErrors?: Record<string, string[]> }`) **or** calls `redirect()` (the success — the module-29 step 5).
4. The hook **sets the state** (the re-render: the form's fields show the errors; the *submitted values* are preserved — the hook *re-binds* the form from the *last submitted FormData* (the *values survive the failed submit* — the module-31's *win over the searchParams pattern*: the *no re-typing*).
5. **If the action threw** (a real error — the module-17 `AppError` 403/404/409, *not* a validation failure): the hook's `isPending` clears, the state *doesn't change* (the *previous* state stays — the *form's* errors remain), and the error is *handled* by the **error boundary** (the module-27 rule 4 — the form's *segment* error) *or* the action *catches* the `AppError` and *returns* it as the state (the module-31's choice: **the action catches the *expected* errors (the 409 "already cancelled", the 422 validation) and *returns* them as state; the *unexpected* (500) *propagates* to the error boundary (the module-27 §4.2 — the digest, the retry) — the *split is the module's rule** (expected = the *state*; unexpected = the *boundary*).

**The JS-off floor** (module 30's path, the *error* version): the *native POST* → the action → the *validation failure* → **no `redirect` to a success page** — the action *must* respond to the *native* POST *somehow*: the **303 back to the same page with `?error=<code>`** (the module-30's §4.1 — the *code*, not the message) — the *page* (the server) reads the code, renders the *summary* error (the *form-level* message — the *field errors are lost* for JS-off (the *honest* degradation: the *JS-off* user sees *"Some fields are invalid — check your input"* (the *summary*) and *re-types* (the *values are gone* (the 303's empty form) — the *module-31's standing line on the floor: **the JS-off error path is *summary-only* (the `?error=` code → the form-level message) — the *field-level* round-trip is the *JS-on* enhancement (the `useActionState` state) — the *floor degrades *gracefully* (the *checkout still *works* (the *no validation is *client-side-only* (the *module-04's* line: the *client Zod is *UX*; the *server Zod is *truth*) — the *JS-off* user hits the *server's* truth (the *303 + the summary*) — the *correctness is *preserved* (the *no invalid data is *stored*).

## 2. Mental Model — The `FormState` DTO (the wire contract for errors)

`FILE: src/features/catalog/product-form-state.ts` (production pattern — [BOTH — BOUNDARY] (imported by the *action* (server) *and* the *form island* (client)))

```ts
// The FormState: the module-14 DTO rules apply (plain-serializable, the *down* direction):
export type ProductFormState = {
  error?: string                       // the form-level message (user-facing — the module-19-03: no internals)
  fieldErrors?: Record<string, string[]>   // field name → messages (the *name is the FormData's name* (the 'price', the 'sku') — the module-31's mapping key)
}
export const initialProductFormState: ProductFormState = {}   // the useActionState's initial (the no-error state)
```

**The mapping rules** (the Zod issue → the field error, the module-04's Zod-4 shapes):

| Zod output (the `safeParse` failure) | The `FormState` shape | The rule |
|---|---|---|
| `error.issues` — `issue.path = ['price']`, `issue.message` | `fieldErrors.price = [issue.message]` | The *path's first element* is the *FormData name* (the *flat schema* — the *module-04's* *Zod* — the *form fields are *flat* (the *`z.object({ name, price, sku, description })`* — the *no nested* paths (a *nested* path (`address.line1`) is the *module-30's checkout* (the *flat* *form field* `name="line1"` — the *schema is *flat* to match the *FormData* (the *module-31's* rule: **the schema's keys *are* the FormData's names** (the *one* *naming* (the *module-13's* design system's *field name* = the *schema key* = the *error key* — the *triple match*)) |
| Multiple issues, *same* field (`price`: the `min` + the `max`) | `fieldErrors.price = ['…', '…']` (the *array* — the *all* messages (the *module-13's* rendering: the *first* + a "and N more" (the *the* *no list of 5* (the *module-13's* error-copy rule))) | The *array* is the *contract* (the *rendering* picks (the *module-13's* `FieldError` renders the *first* + the count)) |
| A *refinement* failure (the `sku` *must be unique* — the *async* check — the *module-17's* *service* (the *Zod's* `superRefine` *async* (the *module-04's* Zod-4 — the *the* *uniqueness is *a *service query* (the *not* the *schema's* *sync* check (the *the* *the schema is *shape*; the *service is *truth* (the *module-17's* line) — the *refinement* *calls* the *service* (the *the* *the action's* *step 2.5* (the *validation's* *async* part — the *module-31's* *two-part validation*: the *sync* shape (the *Zod* `parse`) + the *async* truth (the *service's* uniqueness/ownership — the *module-17's* home) | `fieldErrors.sku = ['SKU already exists']` | The *async* validation is *in the action* (the *after* the *sync parse* (the *module-29's* step 2 *extended*: the *2a* the *shape* (the *Zod*) — the *2b* the *truth* (the *service's* *check* — the *module-17's* *`assertSkuUnique(orgId, sku)`* (the *throws* the *AppError* 422 (the *the* *the action *catches* it (the *expected*) → the *state* (the *field error* on the *sku*) — the *split* (the *module-1's* rule: expected → the *state*) |
| A *form-level* failure (the *no field* is *wrong* — the *org has hit the *product limit* (the *module-11's* *RBAC* — the *plan's* cap)) | `error = 'Your plan allows up to 50 products. Upgrade to add more.'` (the *form-level* message — the *module-13's* copy (the *the* *no field* is *blamed* (the *the* *the limit is *a *form* concern*)) | The *`error`* (the *not* the *`fieldErrors`*) — the *FormError banner* (the *module-30's* §4's `<FormError />`) |

**The `AppError` → state mapping** (the module-17's typed errors, the *expected* set — the action's *catch*):

```ts
// the module-17's AppError (the status, code, message, fieldErrors — module 17's §4):
const EXPECTED_CODES: Record<string, (e: AppError) => ProductFormState> = {
  'validation':       (e) => ({ error: e.message, fieldErrors: e.fieldErrors }),   // the 422 (the *service's* *shape* validation — the *rare* (the *module-04's* line: the *shape is *the action's* (the *the* *the service's 422 is the *defense-in-depth* (the *the* *the action's Zod is the *primary*)
  'already-exists':   (e) => ({ fieldErrors: { [e.field ?? '']: [e.message] } }),  // the 409 (the *uniqueness* — the *2b* truth — the *e.field* is the *module-17's* *AppError's* *field* (the *the* *the service *knows* which field *conflicted*)
  'limit-reached':    (e) => ({ error: e.message }),                                 // the 403-plan (the *form-level*)
  'forbidden':        (e) => ({ error: 'You do not have permission to do that.' }),  // the 403 (the *module-11-03* — the *no internals* (the *module-19-03*) — the *generic* message (the *the* *the specific is *the *server log* (the *module-21's* digest))
  'not-found':        (e) => ({ error: 'That resource no longer exists.' }),         // the 404 (the *stale form* (the *the* *the product was *deleted* while the *form was *open* (the *module-25's* *the* *stale-edit* case))
}
// the *action's* catch (the *module-1's* split):
try { await createProductService(orgId, parsed) }
catch (e) {
  if (isAppError(e) && EXPECTED_CODES[e.code]) return EXPECTED_CODES[e.code](e)   // the *expected* → the *state* (the *round-trip*)
  throw e   // the *unexpected* (the 500 — the *DB down*) → the *propagate* (the *error boundary* (the module-27 §4.2 — the *digest*)
}
```

## 3. Architecture — The form, complete, with the round-trip (the module-30 form, *wired*)

The module-30's `ProductForm` becomes a **client island** (the `useActionState` is a *hook* — the module-30's "the island is the polish" — the *validation round-trip* is *the reason* the form is client (the *the* *no round-trip* form is *server* (the *module-30's* floor *first*) — the *with round-trip* form is *client* (the *the* *the floor *still works* (the *native POST* — the *module-30's* line: the *client form's* *floor is *the same* (the *`action` prop in a client form* — the *module-29's* invocation #1: "forms in Server *and* Client components") — the *island is *the form* (the *module-13's* "the smallest island" — the *the* *the form *is* the *unit* (the *module-31's* architecture: **the form-with-round-trip is *a client island*; the *fields inside it are *server components* (the *module-13's* design system's `TextField` — the *no 'use client'* on the *field* (the *the* *the field is *a *controlled-by-the-form* (the *the* *the form's `defaultValue` + the *state's* `fieldErrors` — the *the* *the field is *presentational* (the *module-13's* line: the *field *renders* the *error* (the *the* *the no *logic* in the *field* — the *the* *the form *owns* the *state* (the *module-13's* the *smallest island*)**):

`FILE: src/features/catalog/components/product-form.tsx` (production pattern — **[CLIENT]** (the form) + the *fields* (server) — the module-31's complete round-trip)

```tsx
'use client'

import { useActionState } from 'react'
import { redirect } from 'next/navigation'   // NOT used here — the *action* redirects (the *the* *the import is *the type* (the *the* *the form *doesn't* redirect (the *the* *the action does*)
import { createProduct, updateProduct } from '../product-actions'
import { initialProductFormState, type ProductFormState } from '../product-form-state'
import { TextField } from '@/components/ui/text-field'     // the module-13's design system (the [SERVER] field — the no 'use client')
import { FormError } from '@/components/ui/form-error'     // the form-level banner (the [SERVER] — the role="alert")
import { SubmitButton } from './submit-button'             // the module-30's §4.2 (the useFormStatus — the *pending*)

export function ProductForm({ product, defaultAction }: { product?: ProductDto; defaultAction?: 'edit' }) {
  // the useActionState: the *state* (the errors) + the *formAction* (the wrapped action — the *module-1's* mechanism step 3-4):
  const action = defaultAction === 'edit' && product
    ? updateProduct.bind(null, product.slug)      // the module-30's §2's static arg (the slug — the *bind* — the *formAction's* version: the *bind on the Server Function reference* (the *the* *the reference's args are *serialized* (the module-14's regime 2 — the *slug is a string*)
    : createProduct
  const [state, formAction] = useActionState(action, initialProductFormState)

  return (
    <form action={formAction} className="space-y-4" noValidate>
      {/* noValidate: the *browser's* validation is *off* (the *module-31's* rule: the *server's* Zod is the *truth* (the *the* *the browser's `required`/`min` would *block* the submit *client-side* (the *the* *the no round-trip* (the *the* *the user can't see the *server's* *specific* error (the *the* *the module-04's line: the *client validation is *UX* (the *optional*) — the *capstone* *chooses* the *server-only* (the *the* *the one source of truth* (the *the* *the noValidate is the *discipline*) — the *required attribute is *kept* (the *the* *the a11y* (the *module-17's* the *`aria-required`* — the *the* *the noValidate *doesn't* remove the *a11y* (the *the* *the field is *marked* (the *the* *the the screen reader *knows* (the *module-17's* a11y line)))) */}
      <fieldset className="space-y-4">
        <legend className="text-sm font-medium">Product</legend>
        <TextField
          name="name" label="Name" required
          defaultValue={product?.name}
          error={state.fieldErrors?.['name']}     // the module-13's TextField's error prop (the [SERVER] field — the *renders* the *aria-invalid* + the *aria-describedby* (the module-17's a11y)
        />
        <TextField
          name="price" label="Price (USD)" type="number" inputMode="decimal" step="0.01" min="0" required
          defaultValue={product ? String(product.price) : undefined}
          error={state.fieldErrors?.['price']}
        />
        <TextField name="sku" label="SKU" defaultValue={product?.sku} error={state.fieldErrors?.['sku']} />
        <div>
          <label htmlFor="description" className="text-sm">Description</label>
          <textarea id="description" name="description" rows={4} defaultValue={product?.description}
                    aria-invalid={state.fieldErrors?.['description'] ? true : undefined}
                    aria-describedby={state.fieldErrors?.['description'] ? 'description-error' : undefined}
                    className="…" />
          {state.fieldErrors?.['description'] && (
            <p id="description-error" className="mt-1 text-sm text-danger">{state.fieldErrors['description'][0]}</p>
          )}
        </div>
      </fieldset>

      {/* the form-level error (the module-13's banner — the role="alert" (the *module-17's* a11y: the *announced* on *render* (the *the* *the the screen reader *reads it* (the *module-17's* line))): */}
      <FormError message={state.error} />

      <div className="flex items-center gap-3">
        <SubmitButton>{product ? 'Save changes' : 'Create product'}</SubmitButton>
      </div>
    </form>
  )
}
```

**The action, complete** (the module-29's five steps + the module-31's validation + the catch — the `createProduct`, the full):

`FILE: src/features/catalog/product-actions.ts` (production pattern — [SERVER])

```ts
'use server'

import { headers } from 'next/headers'
import { redirect } from 'next/navigation'
import { revalidateTag, updateTag } from 'next/cache'
import { z } from 'zod'
import { auth } from '@/lib/auth'
import { createProductService, assertSkuUnique } from '@/services/catalog'
import { TAGS } from '@/lib/tags'
import { AppError, isAppError } from '@/lib/errors'
import type { ProductFormState } from './product-form-state'

// the module-04's shared schema (the *one* schema — the *client* *could* import it (the *UX*) — the *action* *uses* it (the *truth*) — the *module-04's* "Zod vs TS" — the *shape is *the schema*):
const productInput = z.object({
  name: z.string().min(1, 'Name is required').max(120, 'Name must be 120 characters or fewer'),
  price: z.coerce.number({ message: 'Price must be a number' }).min(0, 'Price must be 0 or more').max(999999, 'Price must be 999,999 or less'),
  sku: z.string().min(1).max(40).regex(/^[A-Z0-9-]+$/i, 'SKU: letters, numbers, dashes only').optional().or(z.literal('')),
  description: z.string().max(2000, 'Description must be 2000 characters or fewer').optional().or(z.literal('')),
})

// the useActionState's action signature: (prevState, formData) → the new state (the module-1's mechanism):
export async function createProduct(_prevState: ProductFormState, formData: FormData): Promise<ProductFormState> {
  // 1 — the SESSION gate (the module-29's step 1 — the *action's* job (the *not* the *layout's*):
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) {
    throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })   // the *unexpected* for the *form* (the *the* *the form's *user is *authenticated* (the *the* *the 401 is *the *stale session* (the *module-10's* line) — the *propagate* (the *the* *the error boundary* (the *the* *the module-27 §4.2)) — the *honest*: the *401 in a form* is *the session expired* (the *the* *the the *state* *can't* handle it (the *the* *the redirect to /login is *the *right* response (the *module-10's* line) — the *action* *could* `redirect('/login?from=/products/new')` (the *module-09-04's redirect*) — the *capstone's* choice: the *401 throws* (the *boundary* shows the *"session expired"* (the *module-27's* error) — the *the* *the user re-authenticates* (the *module-10's* flow)
  }
  const orgId = session.organizationId

  // 2a — the VALIDATION: the SHAPE (the Zod — the module-29's step 2 — the *module-31's* 2a):
  const parsed = productInput.safeParse(Object.fromEntries(formData))   // the *FormData → object* (the *module-31's* mapping: the *keys are the FormData's names* (the *module-2's* rule))
  if (!parsed.success) {
    // the *module-2's* mapping: the *issues → the fieldErrors (the *path[0] is the field*):
    const fieldErrors: Record<string, string[]> = {}
    for (const issue of parsed.error.issues) {
      const field = String(issue.path[0] ?? '')
      ;(fieldErrors[field] ??= []).push(issue.message)
    }
    return { error: 'Please fix the highlighted fields.', fieldErrors }   // the *module-1's* step 3 (the *return* the state (the *no redirect* — the *round-trip*)
  }

  // 2b — the VALIDATION: the TRUTH (the service's async checks — the module-17's home — the *module-31's* 2b):
  try {
    if (parsed.data.sku) await assertSkuUnique(orgId, parsed.data.sku)   // the *module-17's* service (the *the* *the uniqueness is *a *query* (the *the* *the throws the AppError 409 'already-exists' (the *e.field = 'sku'*)
  } catch (e) {
    if (isAppError(e) && e.code === 'already-exists') {
      return { fieldErrors: { sku: [e.message] } }   // the *expected* → the *state* (the *module-1's* split)
    }
    throw e
  }

  // 3 — the SERVICE (the module-29's step 3 — the *create* — the *module-17's* DTO-out):
  let created: { slug: string }
  try {
    created = await createProductService(orgId, parsed.data)
  } catch (e) {
    if (isAppError(e) && e.code === 'limit-reached') {
      return { error: e.message }   // the *form-level* (the *module-2's* table's row 4)
    }
    throw e   // the *unexpected* (the 500) → the boundary
  }

  // 4 — the INVALIDATION (the module-29's step 4 — the module-23's spec):
  updateTag(TAGS.products)
  updateTag(TAGS.catalog)
  revalidateTag(TAGS.products, 'max')
  revalidateTag(TAGS.catalog, 'max')

  // 5 — the RESPONSE (the module-29's step 5 — the *303* (the *module-30's* refresh correctness)):
  redirect(`/products/${created.slug}`)   // the *module-1's* mechanism: the *redirect is *not a *returned state* (the *the* *the useActionState *sees* the *NEXT_REDIRECT* (the *module-29's §4's line: the *redirect is *not an error* (the *the* *the the *state* *must not* render an error for it (the *module-5's* mistake: the *redirect-as-error*))
}
```

**The `redirect`-as-state trap** (the module-29's §4, the *form's* version — the *module-5's* row): `useActionState` + the `redirect()` — the *redirect throws* (the `NEXT_REDIRECT` — the module-09-04) — *inside* the action — the *hook* *catches* it (the *framework* handles it (the *the* *the navigation happens* (the *the* *the state is *irrelevant* (the *the* *the no re-render of the *form* (the *the* *the page changes*) — the *trap is the *error boundary* (the *module-27 rule 4): a *naive* `error.tsx` that *renders* "Something went wrong" *would* fire on the *redirect* (the *throw*) — the *module-09-04's* line: the *`error.digest` is *the incident ID* — the *redirect's* *error* has a *known digest* (the `NEXT_REDIRECT` — the *module-09-04's redirect mechanics) — the *capstone's* `error.tsx` (the module-27 §4.2) *checks* the digest (the *not a NEXT_REDIRECT* → the *real error* (the *module-09-04's* verified behavior: the *redirect's error is *distinguishable*) — the *module-5's* fix: the *error.tsx* *ignores* the redirect digest (the *the* *the no false error banner on the *successful submit*).

## 4. Production Code — The JS-off floor's error path (the module-30's `?error=`, the *server* side)

The *native POST* (the JS-off) → the action → the validation failure → the *action* *must respond* (the *no `useActionState` in the *JS-off* (the *the* *the hook is *client*) — the *action's* *dual path* (the *module-31's* architecture: **the action serves *both* paths** (the *JS-on: the *returned state* (the *round-trip*) — the *JS-off: the *303 + the `?error=` code* (the *summary*) — the *how: the *action detects* (the *the* *the `formData` is *the same* (the *the* *the detection is *the `Content-Type`* (the *the* *the native POST is *`multipart/form-data`* (the *the* *the framework's dispatch is *`multipart`* too (the *the* *the no reliable detection* — the *capstone's* choice: **the action *always* returns the state (the *JS-on*) — *and* the *page's* `?error=` is the *floor's* *summary* (the *module-30's §4.1) — the *action's* *JS-off* response is the *303 to the *same page + the `?error=validation`* (the *module-31's* dual-path action: **on validation failure: the *action checks* "am I being called as a *native* POST?" (the *module-19's* line: the *`headers`* — the *`Sec-Fetch-Mode`* (the *`navigate`* for the *native form POST* (the *module-19's verified header) — the *`empty`/`cors`* for the *framework's dispatch*) — the *if `navigate`*: the *303 + the `?error=validation`* (the *floor*) — the *else*: the *return the state* (the *round-trip*) — the *module-31's* production pattern (the *honest* complexity: the *two paths, one action* (the *module-1's title*))**:

`FILE: src/features/catalog/product-actions.ts` (the *dual-path* validation failure — the *production* version, the *module-4's* action *excerpt*)

```ts
  // 2a — the VALIDATION: the SHAPE (the dual path — the module-1's title: "Two paths, one validation"):
  const parsed = productInput.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    const fieldErrors: Record<string, string[]> = {}
    for (const issue of parsed.error.issues) {
      const field = String(issue.path[0] ?? '')
      ;(fieldErrors[field] ??= []).push(issue.message)
    }
    // the *floor detection* (the module-19's Sec-Fetch-Mode — the *native form POST* is `navigate`):
    const headersList = await headers()
    if (headersList.get('sec-fetch-mode') === 'navigate') {
      // the *JS-off* path: the *303 back* + the *code* (the module-30's §4.1 — the *no message in the URL* (the *module-19-03*)):
      redirect('/products/new?error=validation')
    }
    // the *JS-on* path: the *returned state* (the *round-trip* — the *values preserved* (the module-1's step 4)):
    return { error: 'Please fix the highlighted fields.', fieldErrors }
  }
```

**The page's floor rendering** (the module-30's §4.1, the *error* version — the *server* reads the code, *renders* the summary — the *module-4's* edit page's pattern, the *new* page's version):

```tsx
// src/app/(app)/products/new/page.tsx (the *floor's* error — the module-30's §4.1's pattern):
export default async function NewProductPage({ searchParams }: { searchParams: Promise<Record<string, string>> }) {
  const sp = await searchParams
  const errorCode = (sp.error === 'validation' ? 'validation' : undefined)   // the *whitelist* (the *module-19's line: the *code is *validated* (the *the* *the no arbitrary `?error=<anything>` (the *the* *the the *mapping is *closed* (the *module-4's* module-4.1's line: the *code → the message* is *a *closed map*)))
  return (
    <div className="max-w-xl space-y-6">
      <h1 className="text-2xl font-semibold">New product</h1>
      {errorCode === 'validation' && <FormError message="Some fields are invalid — check your input and try again." />}   // the *floor's* summary (the *module-1's* standing line: the *JS-off is *summary-only*)
      <ProductForm />
    </div>
  )
}
```

## 5. Common Mistakes (the validation failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The client-only validation** (the `required`/`min` *without* the `noValidate` — the *browser blocks* the submit; *or* the client Zod *without* the server Zod (the module-04's line *violated*)) | The *JS-off* user is *unvalidated* (the *native POST* → the *no client check* → the *server's* truth (the *correct* (the *module-1's* line) — the *the* *the real bug is the *client Zod *without* the server* (the *the* *the the *server's* *no validation* (the *the* *the the *attacker's* *direct POST* (the module-29's §6) *bypasses* the *client* (the *module-04's* line: **the server Zod is the truth; the client Zod is UX (optional)** — the capstone: *server-only* (the `noValidate`) — the *one source* | The *action's* Zod (the *module-4's* action's step 2a) — the *`noValidate`* (the *browser's* check *off*) — the *client Zod* is *optional UX* (the *module-13's* inline hint (the *debounced* — the *module-10's commit pattern, the *hint's* version — the *not the *gate*)) |
| **The `issue.message` echoed *with* the input** (the *Zod's* message: `"Invalid email: " + input` (the *module-04's* Zod-4's *default message is *safe* (the *no input echo*) — the *custom message* that *echoes* (the *module-19-03's* "no internals" + the *module-17's a11y*: the *error *renders* the *input* (the *XSS* via the *error* — the *RSC's auto-escape* (the *module-14's line) — the *danger is the *`dangerouslySetInnerHTML`* in the *error* (the *module-17's a11y's* `aria-describedby` — the *text* (the *no HTML*) — the *module-5's* line: **the field error is *a string* (the *rendered as text* (the *no HTML* (the *no echo of the raw input* (the *the* *the the *Zod's message is *a template* (the *the* *the the *input is *not in the message* (the *module-04's* line: the *message is *the rule* ("Price must be 0 or more") — the *not* the *value* ("Price must be ≥ 0, you gave -5" — the *echo*)))) | The *Zod's* *message is the rule* (the *module-04's shared schema — the *messages are reviewed* (the *module-13's* error copy) — the *no input echo* (the *module-19-03*) — the *rendering is text* (the *module-14's escape*) |
| **The `fieldErrors` keyed by the *schema key* that ≠ the FormData name** (the *module-2's* triple match *broken* — the *schema key* `priceCents` (the *module-17's* DTO's integer cents) — the *FormData name* `price` (the *user's* dollars) — the *error* on `priceCents` (the *field* renders *no error* (the *the* *the the *error is *orphaned*)) | The *validation passes* (the *the* *the user sees *no error* (the *the* *the the *form *re-submits* (the *the* *the the *no round-trip* (the *module-1's* failure) | The *module-2's* rule: **the schema's keys *are* the FormData's names** (the *triple match*: the *field name* = the *schema key* = the *error key*) — the *`z.coerce`* (the *`price`* → the *number* (the *module-4's action's schema — the *`z.coerce.number`* (the *the* *the the *coercion is *the schema's* (the *the* *the the *FormData's string* → the *number* (the *module-4's Zod-4's verified*)) — the *the* *the the *key is *the FormData's* (the *`price`*) — the *coercion is *the *type* (the *not* the *key*) |
| **The `redirect` rendered as an error** (the *module-3's* trap — the *`error.tsx`* fires on the *`NEXT_REDIRECT`* (the *successful submit* shows the *"Something went wrong"*) | The *successful* create *looks failed* (the *the* *the the *user *re-submits* (the *module-30's* double-mutation — the *create* (the *not idempotent* — the *duplicate product*) | The *module-3's* fix: the *`error.tsx`* *ignores* the redirect digest (the *module-09-04's* distinguishable) — the *the* *the the *action's* *successful path is the redirect* (the *module-29's* step 5) — the *the* *the the *error.tsx's* *check is the digest* (the *module-27's §4.2's `useEffect` — the *digest's* *known-set* (the `NEXT_REDIRECT` → the *ignore*)) |
| **The *async* truth-check *in the schema*** (the *`superRefine`* that *queries the DB* (the *module-17's line violated: the *schema is shape; the service is truth*)) | The *schema* is *untsestable* (the *module-20's testing: the *schema's* *test needs a DB* (the *the* *the the *action's* *step 2b is *the service's* (the *module-17's* home) — the *the* *the the *refine* *catches* the *AppError* (the *module-4's action's 2b) | The *module-17's* split: the *2a shape* (the *Zod* — the *pure*) — the *2b truth* (the *service's* `assertSkuUnique` — the *DB* (the *module-17's home*) — the *action* *orchestrates* (the *module-29's* step 2 *extended*) |
| **The *form state* *without* the `prevState`** (the *action's* signature: `(formData) → state` (the *module-1's mechanism's step 2's `prevState`* *missing*) | The *`useActionState`* *rejects* it (the *build error* — the *hook's* *contract*: the *action takes `(state, formData)`*) — the *confusion*: the *module-29's* action (the `(orderId)` — the *event-handler* action) *vs* the *module-1's* action (the `(prevState, formData)` — the *form* action) — the *two signatures* (the *module-31's* line: **the form action's signature is `(prevState: FormState, formData: FormData) → Promise<FormState>`** (the *module-29's* event-handler action's signature is `(…statics) → Promise<void|DTO>`* (the *the* *the the *two are *different* (the *module-33's decision matrix's row: the *form* action vs the *event-handler* action))) | The *module-1's signature* (the *`_prevState`* (the *unused* (the *the* *the the *re-validation* *could* use it (the *the* *the the *no* (the *module-1's* action *ignores* it (the *the* *the the *formData is *the source* (the *the* *the the *prevState is *the hook's* (the *module-1's mechanism's step 2 — the *re-submit* *sees* the *previous state* (the *the* *the the *action *could* *soften* (the *the* *the the *"you still have the price error"* (the *module-13's copy) — the *capstone* *ignores* it (the *the* *the the *simple*) |
| **The *error message* *leaking* the *validation logic*** (the *message*: `"price must be ≤ 999999 (MAX_PRICE constant)"` (the *module-19-03's* "no internals" — the *constant name* (the *the* *the the *attacker's fingerprint* — the *module-30's §6's line: the *message is user-facing*) | The *internal* *leak* (the *module-19-03*) | The *module-13's* error copy (the *user's language* — the *"Price must be 999,999 or less"* (the *no constant name*) — the *the* *the the *log is the internals* (the *module-21's digest — the *server log has the Zod's issue* (the *the* *the the *user sees the copy*) |

## 6. Security Notes

- **The validation is the *action's* (the *module-29's step 2*) — *never* the *layout's*, *never* the *client's*** (the module-04's line, the *action's* version): the *attacker's direct POST* (the module-29's §6) *bypasses* the *form* (the *the* *the the *action's* Zod is the *only* shape check — the *module-5's row 1, the *security* version).
- **The *async truth* is the *service's* tenancy** (the *module-4's 2b* — the `assertSkuUnique(orgId, sku)`): the *uniqueness is *per-tenant* (the *module-11's multi-tenancy — the *org A's* SKU *can* equal the *org B's* (the *module-17's rule 1: the *orgId-first*) — the *the* *the the *assert* *takes* the *orgId* (the *session's* — the *not the arg's*) — the *cross-tenant* test (the *module-11-02*): *org A* creates a product with *org B's* SKU — the *success* (the *per-tenant uniqueness* (the *correct*) — the *module-5's security check: the *uniqueness is *scoped* (the *module-17's line*)*.
- **The *error round-trip* is the *XSS surface*** (the *module-5's row 2, the *security* version): the *field error is *rendered* (the *module-14's escape* — the *RSC's auto-escape* (the *module-14's line) — the *danger is the *`dangerouslySetInnerHTML`* (the *module-17's a11y — the *no HTML in the error* — the *module-13's design system's `FieldError` is *text* (the *the* *the the *module-5's rule: **the field error is a *string* (the *rendered as text* (the *no HTML* (the *no echo*)** — the *module-20's test: the *input* `<script>alert(1)</script>` in the *name* → the *error* *renders it escaped* (the *module-14's regime 1 — the *string* — the *the* *the the *XSS is *impossible* (the *module-14's line*)**.
- **The *rate limit* is the *login form's*** (the *module-10's subject — the *phase 10's* preview here): the *validation *failure* is the *brute-force* vector (the *module-19's line: the *login's* *error round-trip* is the *"Invalid credentials"* (the *the* *the the *no "email not found" vs "wrong password"* (the *module-10-03's user-enumeration) — the *rate limit* (the *module-19's line: the *per-IP + the per-account* (the *the* *the the *module-10's* subject — the *module-31's* preview: the *form's* *error is the *same* for the *not-found* and the *wrong-password* (the *module-10-03's line*)**.

## 7. Performance Notes

- **The *round-trip* is *one* roundtrip** (the *module-1's mechanism*: the *submit* → the *action* → the *returned state* → the *re-render* — the *module-29's* single roundtrip, the *validation's* version: the *no redirect* (the *validation failure* — the *the* *the the *state is *the response* (the *module-29's* "a single response carries data and UI" — the *state is the data* (the *module-1's line) — the *TTFB of the round-trip* is the *action's* server time (the *module-29's* 20–100ms — the *validation is *fast* (the *Zod's* parse is *µs* (the *module-04's line) — the *2b truth* is the *query* (the *module-17's indexed* — the *~5ms*) — the *the* *the the *round-trip's* latency is the *action's* (the *module-18-01's measurement, the *form's* row*).
- **The *JS-off* path is *two* hops** (the *303 + the GET* — the *module-30's §7's line, the *error's* version): the *native POST* → the *303* → the *GET* (the *full page* — the *module-30's line: the *floor is slower* — the *error's* floor is *the same* (the *303 + the GET* (the *the* *the the *summary is *rendered* (the *module-4's page*) — the *honest degradation* (the *module-1's standing line).
- **The *optimistic* is *not here*** (the *module-32's* subject): the *validation's* *round-trip is *synchronous* (the *the* *the the *error is *before* the *mutation* (the *the* *the the *no optimistic* (the *the* *the the *optimistic is *the mutation's* (the *module-32's line) — the *validation's* *state is *the truth* (the *the* *the the *module-32's* optimistic is *the separate concern* (the *module-33's matrix's row: the *validation (the *module-31*) vs the *optimistic (the module-32*) — the *no conflating* (the *module-5's row 7, the *performance* version: the *the* *the the *round-trip's latency is the *validation's* (the *module-29's action time*) — the *the* *the the *optimistic's* latency is the *mutation's* (the *module-32's*) — the *two are separate* (the *module-33's matrix*).

## 8. Exercise

**Beginner.** Build the *product form's round-trip* (the *module-3's complete form* + the *module-4's action* — the *create* only) — the *stub* service (the *module-29's create* — the *`delay(300)`*). **The round-trip test** (the *JS-on*): *submit with an empty name* → the *field error* on the *name* (the *"Name is required"*) — the *value preserved* (the *the other fields* keep their *input* (the *module-1's step 4* — the *no re-typing*) — the *form-level error* ("Please fix the highlighted fields."). *Submit with a price of -5* → the *price error* ("Price must be 0 or more"). *Submit valid* → the *redirect* (the *303* → the *product page* — the *module-30's* floor test, the *success* version). *Screenshot the three* (the *empty-name*, the *-5 price*, the *success redirect* — the *DevTools' Network: the *RSC payload* (the *`RSC: 1`*) for the *failures* (the *the state is the response*) — the *303* for the *success*).

**Intermediate.** The *async truth* (the *module-4's 2b*): the *SKU uniqueness* — the *stub* `assertSkuUnique` (the *`delay(100)`* + the *the fake DB* (the *a `Map`* — the *the existing SKUs*) — *create a product with SKU "ABC-123"* → the *success* — *create another with "ABC-123"* → the *field error* on the *SKU* ("SKU already exists") — the *form-level* is *absent* (the *field error only*) — *the cross-tenant check* (the *module-6's security*: the *org B* creates "ABC-123" (the *org A's* SKU) → the *success* (the *per-tenant uniqueness* — the *module-17's rule 1*). *Document the three* (the *duplicate*, the *cross-tenant success*, the *the per-tenant line*).

**Production.** The *dual-path* verification (the *module-4's dual-path action*): the *JS-on* (the *round-trip* — the *field errors* — the *values preserved*) + the *JS-off* (the *DevTools' disable* — the *submit invalid* → the *303 + the `?error=validation`* → the *summary* rendered (the *module-4's page*) — *the values lost* (the *honest degradation* — the *module-1's standing line*) — *the success* (the *JS-off* valid submit → the *303* → the *product page*). *The floor's error test* (the *Playwright's* *module-20's* e2e: the *`page.addInitScript(() => { /* disable JS */ })`* — the *module-20's technique* — the *floor's e2e*) — *screenshot the two paths* (the *JS-on field errors*, the *JS-off summary*) — the *artifact: the `docs/validation-matrix.md` (the *per-field* *error cases* + the *per-path* *behavior* — the *module-20's test phase's starting point*).

## 9. Architecture Challenge

**Prompt:** The *member invite form* (the *module-29's action inventory's row 7* — the `inviteMember`): the *fields*: `email` + `role` (the *select*: `member`/`admin`) — the *validation*: the *email format* (the *Zod's* `z.email()` (the *module-04's Zod-4*) + the *role* (the *enum*) + the *truth*: the *email is *not already a member* (the *module-4's 2b* — the *service's* check) + the *the inviter's role* can *assign* the *role* (the *module-11's RBAC: the *`member`* can invite *`member`* only; the *`admin`* can invite *`admin`*) — the *error cases*: the *invalid email* (the *field*), the *already a member* (the *field* — the *email*), the *can't assign that role* (the *form-level* — the *the role select is *the field* (the *the* *the the *error is *on the role* (the *field* — the *the* *the the *message is *"You can only invite members with the 'member' role"*)) — the *the invitee's email is *invalid* and *the inviter can't assign the role* (the *two errors* — the *both* — the *the action's* *order* (the *the* *the the *validate the shape first* (the *both fields*) — the *then the truth* (the *already a member*) — the *then the RBAC* (the *role*) — the *the order matters* (the *module-31's line: **the validation's order is *shape → truth → RBAC* (the *the* *the the *no RBAC error for an invalid email* (the *the* *the the *user fixes the email, then sees the RBAC*) — the *the no email leak** (the *module-10-03's user-enumeration — the *already a member* is *a leak* (the *the* *the the *attacker *enumerates* emails via the *invite form* — the *module-6's security* — the *capstone's choice: the *already a member* is *rendered as "This email is already on the team"* (the *the* *the the *leak is *accepted* (the *module-10-03's line: the *team email is *not sensitive* (the *module-11's multi-tenancy — the *org's* members are *known to the org*) — the *the* *the the *alternative* is the *generic "Could not invite"* (the *the* *the the *no leak* — the *worse UX* — the *capstone's documented choice: the *leak is accepted* (the *module-10-03's line: the *enumeration is *per-org* (the *module-11's tenancy — the *attacker is *in the org* (the *the* *the the *member* can *see the members* (the *module-11's RBAC — the *the* *the the *leak is *to a member* (the *the* *the the *acceptable*)**.

Produce: the *form* (the *module-3's pattern* — the *`useActionState`* — the *fields* (the *email*, the *role select*) — the *a11y* (the *module-17's* — the *role select's* `aria-describedby`) — the *the noValidate*) — the *action* (the *module-4's pattern* — the *dual-path* (the *module-4's 2a/2b*) — the *validation order* (the *shape → truth → RBAC*) — the *the 2b's* *already-a-member* (the *service's* check) — the *RBAC's* *can't-assign* (the *module-11's* `canAssignRole(inviterRole, role)`) — the *invalidation* (the *module-23's spec: the *`'members:{orgId}'`* — the *updateTag* (the *actor*) + the *revalidateTag* (the *fleet*) — the *the member list's* *refresh* (the *module-25's dashboard's adjacent* — the *members page's* *section*) — the *303* (the *members page* + the *`?invited=1`* (the *module-25's dashboard's `?cancelled=1`* pattern, the *invite's* version)) — and the *why* (the *validation order* (the *no RBAC error for invalid email*) — the *email leak* (the *documented choice*) — the *role select's* *server-side re-check* (the *module-29's §6: the *role is client-supplied* (the *the* *the the *attacker's direct POST* with `role=admin` (the *the* *the the *action's RBAC is the *truth* (the *module-11's line: the *client's select is *UX*; the *server's RBAC is *truth*) — the *module-31's standing line: **the form's fields are *the client's*; the action's checks are *the truth* — the *module-29's §6's unsafe/safe pair, the *invite's* version).

<details>
<summary>Model answer</summary>
**The form** (the *module-3's pattern* — the *invite*):

```tsx
// src/features/org/components/invite-form.tsx (the [CLIENT] island — the module-3's pattern):
'use client'
import { useActionState } from 'react'
import { inviteMember } from '../org-actions'
import { initialInviteFormState, type InviteFormState } from '../invite-form-state'
import { TextField } from '@/components/ui/text-field'
import { FormError } from '@/components/ui/form-error'
import { SubmitButton } from '@/components/ui/submit-button'

export function InviteForm() {
  const [state, formAction] = useActionState(inviteMember, initialInviteFormState)
  return (
    <form action={formAction} className="space-y-4" noValidate>
      <div className="grid gap-4 sm:grid-cols-2">
        <TextField name="email" label="Email" type="email" inputMode="email" autoComplete="email" required
                   error={state.fieldErrors?.['email']} />
        <div>
          <label htmlFor="role" className="text-sm">Role</label>
          <select id="role" name="role" defaultValue="member"
                  aria-invalid={state.fieldErrors?.['role'] ? true : undefined}
                  aria-describedby={state.fieldErrors?.['role'] ? 'role-error' : undefined}
                  className="…">
            <option value="member">Member</option>
            <option value="admin">Admin</option>
          </select>
          {state.fieldErrors?.['role'] && (
            <p id="role-error" className="mt-1 text-sm text-danger">{state.fieldErrors['role'][0]}</p>
          )}
        </div>
      </div>
      <FormError message={state.error} />
      <SubmitButton>Send invite</SubmitButton>
    </form>
  )
}
```

**The action** (the *module-4's pattern* — the *dual-path* + the *order*):

```ts
// src/features/org/org-actions.ts (the [SERVER] — the module-4's pattern, the invite):
'use server'
// …imports (the module-29's template)
const inviteInput = z.object({
  email: z.email({ message: 'Enter a valid email address' }),   // the module-04's Zod-4's z.email() (the verified)
  role: z.enum(['member', 'admin'], { message: 'Choose a role' }),
})
export async function inviteMember(_prev: InviteFormState, formData: FormData): Promise<InviteFormState> {
  // 1 — the SESSION gate (the module-29's step 1):
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session?.organizationId) throw new AppError({ status: 401, code: 'unauthenticated', message: 'Sign in required' })
  const orgId = session.organizationId

  // 2a — the SHAPE (the module-4's 2a — the *both fields* (the *email* + the *role*) — the *order: the shape first* (the *no RBAC for invalid email*) (the module-9's line)):
  const parsed = inviteInput.safeParse(Object.fromEntries(formData))
  if (!parsed.success) {
    const fieldErrors: Record<string, string[]> = {}
    for (const issue of parsed.error.issues) {
      const field = String(issue.path[0] ?? '')
      ;(fieldErrors[field] ??= []).push(issue.message)
    }
    const headersList = await headers()
    if (headersList.get('sec-fetch-mode') === 'navigate') redirect(`/settings/members?error=validation`)   // the floor (the module-4's dual-path)
    return { error: 'Please fix the highlighted fields.', fieldErrors }
  }

  // 2b — the TRUTH (the module-4's 2b — the *already a member* (the service's check — the module-17's home)):
  try {
    await assertNotMember(orgId, parsed.data.email)   // the module-17's service (the throws AppError 409 'already-member' (the e.field = 'email'))
  } catch (e) {
    if (isAppError(e) && e.code === 'already-member') {
      return { fieldErrors: { email: [e.message] } }   // the module-6's security: the "This email is already on the team" (the documented leak — the per-org, the module-10-03's line)
    }
    throw e
  }

  // 2c — the RBAC (the module-11's canAssignRole — the *the inviter's role* can *assign* the *role* — the *module-9's line: the RBAC is the truth (the client's select is UX)):
  if (!canAssignRole(session.role, parsed.data.role)) {
    return { fieldErrors: { role: [`You can only invite members with the '${canAssignRole(session.role) ?? 'member'}' role`] } }   // the *field error on the role* (the module-9's line: the role select is the field)
  }

  // 3 — the SERVICE (the module-29's step 3 — the invite — the module-17's DTO-out):
  await inviteMemberService(orgId, parsed.data)   // the module-17's service (the creates the pending membership + the invite email (the module-21's email — the Phase 21's subject — the stub here))

  // 4 — the INVALIDATION (the module-23's spec — the 'members:{orgId}' — the actor immediate + the fleet SWR):
  updateTag(TAGS.members(orgId))
  revalidateTag(TAGS.members(orgId), 'max')

  // 5 — the RESPONSE (the module-29's step 5 — the 303 + the ?invited=1 (the module-25's ?cancelled=1 pattern)):
  redirect(`/settings/members?invited=${encodeURIComponent(parsed.data.email)}`)   // the module-19-03's line: the email in the URL (the *the team member's email* (the *not sensitive* (the module-10-03's line) — the *honest* — the *alternative* is the `?invited=1` (the no email) — the *capstone's choice: the email (the UX — the "You invited dana@…" (the *the* *the the *leak is the same as the already-member (the per-org) — the documented)
}
```

**The why** (the *three* decisions, the *module's* lines):
1. **The validation order** (the *module-9's line*): *shape → truth → RBAC* — the *no RBAC error for an invalid email* (the *the user fixes the email, then sees the RBAC*) — the *the order is the UX* (the *the no "two errors at once" (the *the* *the the *sequential* (the *module-13's copy: the *one error at a time* — the *module-31's line: the *validation's order is *shape → truth → RBAC* (the *the* *the the *no RBAC for invalid email*)**.
2. **The email leak** (the *module-6's security, the documented choice*): the *already a member* is *rendered* (the *"This email is already on the team"*) — the *leak is accepted* (the *module-10-03's line: the *team email is not sensitive* (the *per-org* (the *module-11's tenancy*) — the *the attacker is in the org* (the *the* *the the *member* can see the members* (the *module-11's RBAC*) — the *acceptable*) — the *alternative* (the generic "Could not invite") is *worse UX* (the *the* *the the *no leak* — the *no help*) — the *capstone's documented choice: the leak is accepted (the module-10-03's line*) — the *the security review's row* (the module-19's: the *enumeration is per-org (the module-11's tenancy bounds it*)**.
3. **The role's server-side re-check** (the *module-29's §6, the invite's version*): the *role is client-supplied* (the *select's value* — the *attacker's direct POST* with `role=admin` (the *module-29's line: the *reachable by anyone*) — the *action's RBAC is the truth* (the *module-11's line: the *client's select is UX; the server's RBAC is truth*) — the *module-9's line: the RBAC is the truth (the client's select is UX*) — the *module-31's standing line: the form's fields are the client's; the action's checks are the truth* (the *module-29's §6's unsafe/safe pair, the invite's version*) — the *the cross-tenant/role test* (the *module-11-02's*: a *member* POSTs `role=admin` → the *field error on the role* (the *the RBAC's denial* — the *module-11-03's 403-as-field-error* (the *the RBAC's denial is a *field error* (the *module-31's line: the *RBAC's error is a *field* (the *role*) — the *no form-level* (the *the* *the the *user sees the role's error* (the *module-13's copy*))**.
**The generalization** (the *invite's pattern*, the *module's standing rule*): **a form with *RBAC-gated fields* (the role, the plan, the permission) is *the form* (the fields — the client's) + *the action's RBAC* (the truth — the module-11's `canAssignRole`) — the *validation order is shape → truth → RBAC* (the no RBAC for invalid shape) — the *RBAC's error is a field error* (the gated field — the module-13's copy) — the *the leak is documented* (the module-10-03's line: the *per-org enumeration is acceptable*) — the *the module-31's standing line: the form's fields are the client's; the action's checks are the truth* (the module-29's §6's unsafe/safe pair, the invite's version)**.
</details>

## 10. Official Documentation

- `useActionState` (the React 19 hook — the round-trip): https://react.dev/reference/react/useActionState
- Forms (the action's returned state, the field errors): https://nextjs.org/docs/app/guides/forms
- Zod 4 (the `z.email`, the `z.coerce`, the `safeParse`): https://zod.dev/
- Handling Errors (the error boundaries, the `error.digest`): https://nextjs.org/docs/app/guides/handling-errors
- Server Actions and Mutations (the security, the direct POST): https://nextjs.org/docs/app/guides/server-actions
- `redirect` (the 303, the `NEXT_REDIRECT`): https://nextjs.org/docs/app/api-reference/functions/redirect

## 11. What You Should Know Before Continuing

- [ ] I can state the *round-trip's mechanism* (the `useActionState` — the `(prevState, formData) → state` — the *values preserved* — the *no re-typing*) — the *module-1's mechanism's 5 steps*
- [ ] I know the *two paths, one validation* (the *JS-on: the returned state* (the field errors + the values) — the *JS-off: the 303 + the `?error=` code* (the summary-only — the *honest degradation*) — the *dual-path action* (the `sec-fetch-mode` detection)
- [ ] I can write the *`FormState` DTO* (the *`error`* + the *`fieldErrors`* — the *module-2's mapping: the Zod issue → the field* — the *triple match: the field name = the schema key = the error key*)
- [ ] I know the *validation order* (the *shape (2a) → truth (2b) → RBAC (2c*) — the *no RBAC for invalid shape*) — and the *expected/unexpected split* (the *expected → the state*; the *unexpected → the boundary*)
- [ ] I know the *`redirect`-as-state trap* (the `NEXT_REDIRECT` — the `error.tsx` ignores it — the *no false error on success*)
- [ ] I know the *security lines* (the *server Zod is the truth*; the *field error is text, no echo, no HTML*; the *RBAC is the action's*; the *leak is documented*)
- [ ] I've done the *round-trip test* (the *empty name*, the *-5 price*, the *success redirect* — the *screenshot the three*) and the *dual-path verification* (the *JS-on field errors*, the *JS-off summary*)

**Next:** Module 32 — Optimistic UI & Pending States (the `useOptimistic` + the `useFormStatus` + the rollback on server error — the *perceived* instantaneity — the *optimistic's* label ("requested," not "done") — the *module-32's decision table* (the *per-mutation* optimistic)).
