# Module 47 — Auth with Cache Components: The Session Is a Private Read, the Data Is a Tagged Cache

**Phase 10: Authentication · Module 47 of 101**

> **Where does this run?** Everything is **`[SERVER]`** — but the *session read* is a **request-time** read (module 47's §1) that **can't be in the static shell** (module 27's shell, module 47's §1) — it streams behind a `<Suspense>` hole (module 27's hole). The *session-derived data* is a **`[SERVER]` cache** (the `use cache` + the `cacheTag`, module 21's/23's). The module-47's standing rule (the verified official guide, the module-47's §1): **the session read is `use cache: private` (browser-only, never server-cached) — the session-derived data is `use cache` keyed by the extracted id (tagged, invalidated)** — the *module-47's line: the session is the private, the data is the tagged* (module 47's §1).

---

## 1. Concept — The session is a request read (the shell's hole)

**The session read is the request-time** (module 47's §1): the *the `cookies()`/`headers()` is the *request* (module 47's §1) — the *the no prerender* (module 47's §1) — the *module-47's line: the session read is the request-time* (module 47's §1) — the *module-27's line: the shell is the static* (module 27's) — the *module-47's line: the session is the shell's hole* (module 47's §1).

**The `use cache: private`** (module 47's §1.1): the *the directive for the session* (module 47's §1.1) — the *the `cookies()` + the `headers()` + the `searchParams`* (module 47's §1.1) — the *the no `use cache`/`use cache: remote`* (module 47's §1.1) — the *module-47's line: the `use cache: private` is the session's* (module 47's §1.1) — the *the browser-only* (module 47's §1.1) — the *the no server's cache* (module 47's §1.1).

**The `use cache`'s derived** (module 47's §1.2): the *the `use cache` is the *data's* (module 47's §1.2) — the *the id's argument is the *cache key* (module 47's §1.2) — the *module-47's line: the `use cache` is the data's* (module 47's §1.2) — the *the `cacheTag` is the invalidation's* (module 47's §1.2) — the *the `updateTag` is the refresh's* (module 23's).

**The module-20-26's doctrine, reconciled** (module 47's §1.3): the *the module-20-26's line: the session+orgId is the NO CACHED* (module 20-26's) — the *module-47's line: the session's read is the NO server's cached (module 47's §1.1) — the *the session's *derived's* is the CACHED (module 47's §1.2) — the *module-47's line: the session's read is the no-cached, the derived's is the cached* (module 47's §1.3) — the *module-20-26's line: the session+orgId is the no-cached* (module 20-26's) — the *module-47's line: the session's derived's is the cached* (module 47's §1.2).

## 2. Mental Model — The 2 reads (drawn)

```mermaid
flowchart TD
    A["the REQUEST (module 2's)"] --> B{"the WHAT?"}
    B -->|the session (the who)| C["the use cache: private (module 47's §1.1) — the browser-only — the NO server's cache"]
    B -->|the session-derived's data (the what)| D["the use cache (module 47's §1.2) — the id's argument (the cache key) — the cacheTag (the invalidation)"]
    C --> E["the <Suspense>'s hole (module 27's) — the stream's"]
    D --> E
    E --> F["the HTML (module 2's)"]
```

**The 2 reads** (the module-47's mental model):
1. **The session's read** (module 47's §1.1): the *the `use cache: private`* — the *the browser-only* — the *the no server's cache* (module 47's §1.1).
2. **The derived's read** (module 47's §1.2): the *the `use cache`* — the *the id's argument* — the *the `cacheTag`* (module 47's §1.2).

## 3. Architecture — The session read (the `use cache: private`, the code)

`FILE: src/lib/session.ts` (production pattern — [SERVER] — the module-47's §3: the `getCurrentUser`, the `use cache: private`)

```ts
// THE SESSION READ (module 47's §3 — the use cache: private (module 47's §1.1) — the the module-43's §4's single entry point (module 43's §4)):
import 'server-only'
import { headers } from 'next/headers'
import { redirect } from 'next/navigation'
import { auth } from '@/auth'

export type User = {
  id: string
  name: string | null
  email: string
}

export async function getCurrentUser(): Promise<User> {
  'use cache: private'   // THE use cache: private (module 47's §1.1) — the the browser-only (module 47's §1.1) — the the no server's cache (module 47's §1.1)

  const s = await auth.api.getSession({ headers: await headers() })   // the module-44's §3.6 (module 44's) — the the request-time (module 47's §1)
  if (!s) redirect('/login')   // the module-47's line: the redirect is the no-cached (module 47's §3.1) — the the throw's (module 47's §3.1)
  return { id: s.user.id, name: s.user.name, email: s.user.email }   // the module-47's line: the narrow's DTO (module 47's §3.1) — the the no raw's (module 47's §3.1)
}
```

**The module-47's line:** the *session read is the `use cache: private`* (module 47's §1.1) — the *the browser-only* (module 47's §1.1) — the *the `redirect` is the no-cached* (module 47's §3.1) — the *the narrow's DTO* (module 47's §3.1).

### 3.1 The `redirect`'s no-cached (module 47's §3.1)

- **The `redirect()` is the throw** (module 47's §3.1): the *the `redirect()` is the *throw* (module 47's §3.1) — the *the no value's return* (module 47's §3.1) — the *module-47's line: the `redirect` is the no-cached* (module 47's §3.1) — the *the only the resolved's user is the cached* (module 47's §3.1).

**The module-47's line:** the *`redirect` is the no-cached* (module 47's §3.1) — the *the only the resolved's user is the cached* (module 47's §3.1).

## 4. Production Code — The page (the 2 reads, the code)

`FILE: src/app/dashboard/page.tsx` (production pattern — [SERVER] — the module-47's §4: the shell + the hole)

```tsx
// THE PAGE (module 47's §4 — the shell (module 27's) + the hole (module 27's) — the the 2 reads (module 47's §2)):
import { Suspense } from 'react'
import { getCurrentUser } from '@/lib/session'
import { getDashboardData } from '@/lib/dashboard'   // the module-47's §5 (module 47's §5) — the the derived's (module 47's §1.2)

export default function DashboardPage() {
  return (
    <main>
      {/* THE SHELL (module 27's): the the no session's read (module 47's §1) — the the static's (module 27's) */}
      <p>Dashboard</p>

      {/* THE HOLE (module 27's): the the session's read (module 47's §1) — the the stream's (module 27's) */}
      <Suspense fallback={<p>Loading your dashboard…</p>}>
        <Dashboard />
      </Suspense>
    </main>
  )
}

async function Dashboard() {
  const user = await getCurrentUser()   // the module-47's line: the session's read is the private (module 47's §1.1)
  const data = await getDashboardData()   // the module-47's line: the derived's is the cached (module 47's §1.2)
  return <h1>Welcome, {user.name}</h1>
}
```

**The module-47's line:** the *page is the shell + the hole* (module 27's) — the *the session's read is the private* (module 47's §1.1) — the *the derived's is the cached* (module 47's §1.2).

## 5. Architecture — The derived's data (the `use cache` + the `cacheTag`, the code)

`FILE: src/lib/dashboard.ts` (production pattern — [SERVER] — the module-47's §5: the 2 functions, the exported + the unexported)

```ts
// THE DERIVED'S DATA (module 47's §5 — the 2 functions (module 47's §5) — the the exported's + the unexported's (module 47's §5.1)):
import 'server-only'
import { cacheLife, cacheTag } from 'next/cache'
import { getCurrentUser } from './session'
import { db } from '@/db'
import { products } from '@/db/schema'
import { eq } from 'drizzle-orm'

// THE EXPORTED'S (module 47's §5): the the resolve's user (module 47's §5) — the the no id's argument (module 47's §5.1):
export async function getDashboardData() {
  const user = await getCurrentUser()   // the module-47's line: the resolve's user is the exported's (module 47's §5) — the the private's (module 47's §1.1)
  return getDashboardDataByUserId(user.id)   // the module-47's line: the id's argument is the unexported's (module 47's §5.1)
}

// THE UNEXPORTED'S (module 47's §5.1): the the id's argument is the *cache key* (module 47's §1.2) — the the no caller's id (module 47's §5.1):
async function getDashboardDataByUserId(userId: string) {
  'use cache'   // THE use cache (module 47's §1.2) — the the server's cache (module 47's §1.2)
  cacheTag(`dashboard:${userId}`)   // the module-47's line: the cacheTag is the invalidation's (module 47's §1.2) — the the id's key (module 47's §1.2)
  cacheLife('minutes')   // the module-22's line: the cacheLife is the profile's (module 22's) — the the minutes' (module 47's §5.1)

  return db.query.products.findMany({
    where: eq(products.orgId, /* the orgId from the session (module 11's) */ userId),   // the module-17's rule 1 (module 17's) — the the tenancy's scope (module 17's rule 1)
  })
}
```

**The module-47's line:** the *derived's is the 2 functions* (module 47's §5) — the *the exported's is the resolve's user* (module 47's §5) — the *the unexported's is the id's argument* (module 47's §5.1) — the *the `cacheTag` is the invalidation's* (module 47's §1.2) — the *the no caller's id* (module 47's §5.1).

### 5.1 The unexported's id (module 47's §5.1 — the data security)

- **The unexported's is the safe** (module 47's §5.1): the *the no caller's id* (module 47's §5.1) — the *the resolve's user is the exported's* (module 47's §5) — the *module-47's line: the unexported's is the safe* (module 47's §5.1) — the *module-19-02's IDOR* (module 75's) — the *module-47's line: the unexported's is the safe* (module 47's §5.1).

**The module-47's line:** the *unexported's is the safe* (module 47's §5.1) — the *the no caller's id* (module 47's §5.1) — the *module-19-02's IDOR* (module 75's).

## 6. Production Code — The action (the `updateTag`, the code)

`FILE: src/app/dashboard/actions.ts` (production pattern — [SERVER] — the module-47's §6: the 5-step + the `updateTag`)

```ts
// THE ACTION (module 47's §6 — the 5-step (module 29's) + the updateTag (module 23's) — the the module-47's line: the re-read's session is the self's auth (module 47's §6)):
'use server'
import { redirect } from 'next/navigation'
import { updateTag } from 'next/cache'
import { getCurrentUser } from '@/lib/session'
import { db } from '@/db'
import { products } from '@/db/schema'
import { eq } from 'drizzle-orm'

export async function addProduct(formData: FormData) {
  // THE 1: THE SESSION'S GATE (module 29's step 1) — the the re-read's session (module 47's §6) — the the no client's trust (module 47's §6):
  const user = await getCurrentUser()   // the module-47's line: the re-read's session is the self's auth (module 47's §6) — the the no client's id (module 47's §6)
  if (!user) redirect('/login')   // the module-29's step 1 (module 29's)

  // THE 2: THE ZOD (module 29's step 2) — the the shape (module 29's):
  // ... (module 29's step 2's Zod's)

  // THE 3: THE SERVICE (module 29's step 3) — the the write (module 17's):
  await db.insert(products).values({ orgId: user.id, slug: 'new', name: 'New', priceCents: 100, status: 'draft' })   // the module-17's rule 1 (module 17's) — the the tenancy's scope (module 17's rule 1)

  // THE 4: THE INVALIDATE (module 29's step 4) — the updateTag (module 23's):
  updateTag(`dashboard:${user.id}`)   // the module-47's line: the updateTag is the invalidation's (module 47's §1.2) — the the same tag (module 23's)

  // THE 5: THE REDIRECT (module 29's step 5):
  redirect('/dashboard')   // the module-29's step 5 (module 29's)
}
```

**The module-47's line:** the *action is the 5-step + the `updateTag`* (module 47's §6) — the *the re-read's session is the self's auth* (module 47's §6) — the *the no client's id* (module 47's §6) — the *the same tag* (module 23's).

## 7. Common Mistakes (the cache's failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The `use cache`'s session** (module 47's §1.1's line violated) | the *module-47's line: the `use cache: private` is the session's* (module 47's §1.1) — the *the `use cache`'s session is the *build error* (module 47's §1.1) — the *module-47's line: the `use cache: private` is the session's* (module 47's §1.1) — the *no `use cache`'s session* (module 47's §1.1)* | the *the `use cache: private`* (module 47's §1.1) — the *module-47's line: the `use cache: private` is the session's* (module 47's §1.1)* |
| **The no `<Suspense>`** (module 47's §1's line violated) | the *module-47's line: the session is the shell's hole* (module 47's §1) — the *the no `<Suspense>` is the *build error* (module 47's §1) — the *module-47's line: the session is the shell's hole* (module 47's §1) — the *no `<Suspense>`* (module 47's §1)* | the *the `<Suspense>`'s hole* (module 27's) — the *module-47's line: the session is the shell's hole* (module 47's §1)* |
| **The id's argument in the exported's** (module 47's §5.1's line violated) | the *module-47's line: the unexported's is the safe* (module 47's §5.1) — the *the id's argument in the exported's is the *IDOR* (module 47's §5.1) — the *module-19-02's IDOR* (module 75's) — the *module-47's line: the unexported's is the safe* (module 47's §5.1) — the *no IDOR* (module 47's §5.1)* | the *the unexported's id* (module 47's §5.1) — the *module-47's line: the unexported's is the safe* (module 47's §5.1)* |
| **The no `cacheTag`** (module 47's §1.2's line violated) | the *module-47's line: the `cacheTag` is the invalidation's* (module 47's §1.2) — the *the no `cacheTag` is the *stale's* (module 47's §1.2) — the *module-47's line: the `cacheTag` is the invalidation's* (module 47's §1.2) — the *no stale's* (module 47's §1.2)* | the *the `cacheTag`* (module 47's §1.2) — the *module-47's line: the `cacheTag` is the invalidation's* (module 47's §1.2)* |
| **The no `updateTag`** (module 47's §6's line violated) | the *module-47's line: the `updateTag` is the refresh's* (module 23's) — the *the no `updateTag` is the *stale's* (module 23's) — the *module-47's line: the `updateTag` is the refresh's* (module 23's) — the *no stale's* (module 23's)* | the *the `updateTag`* (module 23's) — the *module-47's line: the `updateTag` is the refresh's* (module 23's)* |
| **The raw's session to the client** (module 47's §3.1's line violated) | the *module-47's line: the narrow's DTO* (module 47's §3.1) — the *the raw's session is the *leak* (module 47's §3.1) — the *module-47's line: the narrow's DTO* (module 47's §3.1) — the *no raw's session to the client* (module 47's §3.1)* | the *the narrow's DTO* (module 47's §3.1) — the *module-47's line: the narrow's DTO* (module 47's §3.1)* |
| **The client's id in the action** (module 47's §6's line violated) | the *module-47's line: the re-read's session is the self's auth* (module 47's §6) — the *the client's id is the *spoof* (module 47's §6) — the *module-47's line: the re-read's session is the self's auth* (module 47's §6) — the *no client's id* (module 47's §6)* | the *the `getCurrentUser()`* (module 47's §6) — the *module-47's line: the re-read's session is the self's auth* (module 47's §6)* |
| **The secret in the tag** (module 47's §7.1's line violated) | the *module-47's line: the tag is the no secret* (module 47's §7.1) — the *the secret in the tag is the *leak* (module 47's §7.1) — the *module-47's line: the tag is the no secret* (module 47's §7.1) — the *no secret in the tag* (module 47's §7.1)* | the *the id's key* (module 47's §1.2) — the *module-47's line: the tag is the no secret* (module 47's §7.1)* |

## 8. Security Notes

- **The re-read's session is the self's auth** (module 47's §6): the *module-47's line: the re-read's session is the self's auth* (module 47's §6) — the *the no client's trust* (module 47's §6) — the *module-29's step 1* (module 29's).
- **The unexported's is the safe** (module 47's §5.1): the *module-47's line: the unexported's is the safe* (module 47's §5.1) — the *module-19-02's IDOR* (module 75's) — the *the no caller's id* (module 47's §5.1).
- **The narrow's DTO** (module 47's §3.1): the *module-47's line: the narrow's DTO* (module 47's §3.1) — the *module-19-02's leak* (module 75's) — the *the no raw's session* (module 47's §3.1).
- **The tag is the no secret** (module 47's §7.1): the *module-47's line: the tag is the no secret* (module 47's §7.1) — the *module-19-02's leak* (module 75's) — the *the no secret in the tag* (module 47's §7.1).
- **The tenancy's scope is the first** (module 17's rule 1): the *module-17's line: the tenancy's scope is the first* (module 17's rule 1) — the *module-47's line: the tenancy's scope is the first* (module 17's rule 1) — the *module-11-02's cross-tenant* (module 11-02's).
- **The `taintUniqueValue` is the client's guard** (module 47's §8.1): the *module-47's line: the `taintUniqueValue` is the client's guard* (module 47's §8.1) — the *module-19-02's XSS* (module 75's) — the *the no sensitive's field* (module 47's §8.1).

## 9. Performance Notes

- **The session's read is the private's** (module 47's §1.1): the *module-47's line: the session's read is the private's* (module 47's §1.1) — the *the no server's cache* (module 47's §1.1) — the *module-22's* *the TTFB's* (module 22's).
- **The derived's is the cached's** (module 47's §1.2): the *module-47's line: the derived's is the cached's* (module 47's §1.2) — the *the `cacheLife` is the profile's* (module 22's) — the *module-22's* *deep-dive* (module 22's).
- **The `use cache: remote` is the durable's** (module 47's §9.1): the *module-47's line: the `use cache: remote` is the durable's* (module 47's §9.1) — the *module-21's* *the remote's* (module 21's) — the *module-21's* *deep-dive* (module 21's).
- **The shell is the static's** (module 27's): the *module-27's line: the shell is the static* (module 27's) — the *module-47's line: the shell is the static's* (module 27's) — the *module-27's* *deep-dive* (module 27's).

## 10. Exercise

**Beginner.** *The session's read* (module 47's §3): the *the `getCurrentUser`* (module 3's) + the *the `use cache: private`* (module 3's) + the *the `<Suspense>`'s hole* (module 4's) — *build it* — the *the dashboard's log* (module 20's) — the *artifact: the dashboard's HTML + the log* (module 20's).

**Intermediate.** *The derived's data* (module 47's §5): the *the 2 functions* (module 5's) + the *the `cacheTag`* (module 5's) + the *the `cacheLife`* (module 5's) — the *artifact: the derived's data + the log* (module 20's).

**Production.** *The action's invalidate* (module 47's §6): the *the 5-step* (module 6's) + the *the `updateTag`* (module 6's) + the *the re-read's session* (module 6's) — the *artifact: the action's log + the invalidate's log* (module 20's).

## 11. Architecture Challenge

**Prompt:** The *"the partner's dashboard is stale: the user's order's create is the no refresh"* (the *module-47's* *derived's* — the *module-23's* *invalidate's* — the *module-47's line: the `updateTag` is the refresh's* (module 23's) — the *module-47's standing line: the derived's is the cached's + the `updateTag` is the refresh's* (module 47's §1.2 + module 23's)).

The *problems*: (1) the *the derived's is the cached's* (the *the `use cache`* (module 47's §1.2) — the *module-47's line: the derived's is the cached's* (module 47's §1.2) — the *module-47's standing line: the derived's is the cached's* (module 47's §1.2)).

(2) the *the `updateTag` is the refresh's* (the *the `updateTag`* (module 23's) — the *module-47's line: the `updateTag` is the refresh's* (module 23's) — the *module-47's standing line: the `updateTag` is the refresh's* (module 23's)).

**Design**: the *the invalidate's* (the *the `use cache`* (module 47's §1.2) + the *the `cacheTag`* (module 47's §1.2) + the *the `updateTag`* (module 23's) — the *module-47's line: the derived's is the cached's + the `updateTag` is the refresh's* (module 47's §1.2 + module 23's) — the *module-47's standing line: the derived's is the cached's + the `cacheTag` is the invalidation's + the `updateTag` is the refresh's* (module 47's §1.2 + module 23's)).

Produce: the *the invalidate's* (the *the `use cache`* (module 47's §1.2) + the *the `cacheTag`* (module 47's §1.2) + the *the `updateTag`* (module 23's) — the *module-47's line: the derived's is the cached's + the `updateTag` is the refresh's* (module 47's §1.2 + module 23's) — the *module-47's standing line: the derived's is the cached's + the `cacheTag` is the invalidation's + the `updateTag` is the refresh's* (module 47's §1.2 + module 23's)).

<details>
<summary>Model answer</summary>
**The invalidate's** (module 47's §1.2 + module 23's):
1. **The derived's is the cached's** (module 47's §1.2): the *the `use cache`* (module 47's §1.2) + the *the `cacheTag`* (module 47's §1.2) — the *module-47's line: the derived's is the cached's* (module 47's §1.2).
2. **The `updateTag` is the refresh's** (module 23's): the *the `updateTag`* (module 23's) — the *module-47's line: the `updateTag` is the refresh's* (module 23's).
**The generalization** (the *invalidate's* pattern, the *module's* standing rule): **the *derived's is the cached's* (module 47's §1.2) — the *the `cacheTag` is the invalidation's* (module 47's §1.2) — the *the `updateTag` is the refresh's* (module 23's) — the *module-47's standing line: the derived's is the cached's + the `cacheTag` is the invalidation's + the `updateTag` is the refresh's* (module 47's §1.2 + module 23's)*.
</details>

## 12. Official Documentation

- The Next.js Authentication with Cache Components guide: https://nextjs.org/docs/app/guides/authentication-with-cache-components
- The Next.js Authentication guide: https://nextjs.org/docs/app/guides/authentication
- The Next.js Data Security guide: https://nextjs.org/docs/app/guides/data-security
- The Next.js `use cache: private`: https://nextjs.org/docs/app/api-reference/directives/use-cache-private
- Better Auth: Next.js integration: https://www.better-auth.com/docs/integrations/next
- The module-44's setup: the module-44 (the phase-10's file-02)

## 13. What You Should Know Before Continuing

- [ ] I can state the *session is the request-time* (module 1's) — the *the no prerender* (module 1's line) — the *the shell's hole* (module 1's line)
- [ ] I know the *`use cache: private` is the session's* (module 1.1's) — the *the browser-only* (module 1.1's) — the *the no server's cache* (module 1.1's)
- [ ] I know the *`use cache` is the data's* (module 1.2's) — the *the id's argument is the cache key* (module 1.2's) — the *the `cacheTag` is the invalidation's* (module 1.2's)
- [ ] I know the *derived's is the 2 functions* (module 5's) — the *the exported's is the resolve's user* (module 5's) — the *the unexported's is the id's argument* (module 5.1's)
- [ ] I know the *action is the 5-step + the `updateTag`* (module 6's) — the *the re-read's session is the self's auth* (module 6's) — the *the no client's id* (module 6's)
- [ ] I know the *`redirect` is the no-cached* (module 3.1's) — the *the only the resolved's user is the cached* (module 3.1's)
- [ ] I've done the *session's read* (module 10's beginner) + the *derived's data* (module 10's intermediate) + the *action's invalidate* (module 10's production) — the *artifacts* (module 20's)

**Phase 10 complete.** Next: Phase 11 — Authorization & RBAC (module 48: the RBAC's architecture — module 49: the multi-tenancy — module 50: the 401 vs 403).
