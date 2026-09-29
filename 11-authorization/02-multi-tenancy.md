# Module 49 — Multi-tenancy: orgId From the Session, Never From the Caller

**Phase 11: Authorization & RBAC · Module 49 of 101**

> **Where does this run?** The tenancy is **`[SERVER]`** at every layer (module 38's constraint, module 17's scope, module 49's org context). The *orgId* is a **`[SERVER]` fact** (the session's, module 49's §1) — **never** a client argument (module 49's §1, the module-17's rule 1). This module is the **anchor for `module 11-02`** — the cross-tenant test every other module cites (module 29's action test, module 36's API-key mismatch, module 25's cache-key test, module 11-02's §4). The module-49's standing rule: **orgId comes from the session, never from the caller — and the cross-tenant test is the proof** (module 49's §1).

---

## 1. Concept — The tenancy is the org's (the 3 lines, the 1 rule)

**The rule** (module 49's §1): the *the `orgId` is the session's* (module 49's §1) — the *the no caller's arg* (module 49's §1) — the *module-49's line: the orgId is the session's, never the caller's* (module 49's §1) — the *module-17's rule 1: the orgId-first* (module 17's rule 1).

**The 3 lines** (module 49's §1.1): the *the service's scope* (module 17's rule 1) — the *the query's scope* (module 39's) — the *the DB's constraint* (module 38's) — the *module-49's line: the 3 lines are the service's + the query's + the DB's* (module 49's §1.1) — the *module-19's line: the defense in depth* (module 19's).

**The model** (module 49's §1.2): the *the shared-schema, shared-rows* (module 11's decision) — the *the `org_id` on every tenant-owned row* (module 38's §1) — the *module-49's line: the model is the shared-schema* (module 49's §1.2) — the *module-38's line: the tenancy is the schema* (module 38's §1).

## 2. Mental Model — The org context (the resolution, drawn)

```mermaid
flowchart TD
    A["the REQUEST (module 2's)"] --> B{"the orgId's WHERE? (module 49's §2)"}
    B -->|the route's [orgId]| C["the route's param (module 49's §2.1) — the /orgs/[orgId]/…"]
    B -->|the no route's| D["the active's org (module 49's §2.2) — the user_active_org's row"]
    C --> E["the requireOrgMember (module 49's §2.3) — the membership's check (module 38's)"]
    D --> E
    E -->|the member| F["the orgId (module 49's §1) — the session's fact (module 49's §1)"]
    E -->|the no member| G["the 404 (module 50's) — the no leak (module 50's)"]
```

**The 3 steps** (the module-49's mental model):
1. **The route's** (module 49's §2.1): the *the `/orgs/[orgId]/…`* — the *module-49's line: the route's is the explicit's* (module 49's §2.1).
2. **The active's** (module 49's §2.2): the *the `user_active_org`'s row* — the *module-49's line: the active's is the implicit's* (module 49's §2.2).
3. **The check** (module 49's §2.3): the *the `requireOrgMember`* — the *module-49's line: the check is the membership's* (module 49's §2.3).

## 3. Architecture — The org's resolution (the code)

### 3.1 The route's org (module 49's §2.1 — the `/orgs/[orgId]`)

`FILE: src/services/orgs.ts` (production pattern — [SERVER] — the module-49's §3.1: the `getOrgFromRoute`)

```ts
// THE ROUTE'S ORG (module 49's §2.1 — the /orgs/[orgId] (module 49's §2.1) — the the module-49's line: the route's is the explicit's (module 49's §2.1)):
import 'server-only'
import { requireOrgMember } from '@/lib/rbac'   // the module-49's §3.3 (module 49's §3.3)

export async function getOrgFromRoute(session: { userId: string }, orgId: string) {
  // THE CHECK (module 49's §2.3): the the membership's check (module 49's §3.3) — the the no leak (module 50's):
  await requireOrgMember(session.userId, orgId)   // the module-49's line: the check is the membership's (module 49's §2.3) — the the 404's (module 50's)
  return orgId   // the module-49's line: the orgId is the session's fact (module 49's §1) — the the checked (module 49's §2.3)
}
```

**The module-49's line:** the *route's is the explicit's* (module 49's §2.1) — the *the check is the membership's* (module 49's §2.3) — the *the no leak* (module 50's).

### 3.2 The active's org (module 49's §2.2 — the `user_active_org`)

`FILE: src/db/schema.ts` (the active's org's table — [SERVER] — the module-49's §3.2)

```ts
// THE ACTIVE'S ORG (module 49's §2.2 — the user_active_org (module 49's §3.2) — the the implicit's (module 49's §2.2)):
export const userActiveOrg = pgTable('user_active_org', {
  userId: uuid('user_id').primaryKey().references(() => users.id, { onDelete: 'cascade' }),   // the module-49's line: the (user) is the PK (module 49's §3.2)
  orgId: uuid('org_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),   // the module-49's line: the (org) is the FK (module 49's §3.2)
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
})
```

`FILE: src/services/orgs.ts` (production pattern — [SERVER] — the module-49's §3.2: the `getActiveOrg`)

```ts
// THE ACTIVE'S ORG (module 49's §2.2 — the getActiveOrg (module 49's §3.2) — the the implicit's (module 49's §2.2)):
export async function getActiveOrg(session: { userId: string }) {
  // THE ACTIVE'S READ (module 49's §3.2): the the user_active_org's row (module 49's §3.2):
  const row = await db.query.userActiveOrg.findFirst({
    where: eq(userActiveOrg.userId, session.userId),   // the module-49's line: the (user) is the scope (module 49's §3.2)
  })
  if (!row) return null   // the module-49's line: the no active's is the null (module 49's §3.2) — the the org's select (module 49's §9)
  // THE CHECK (module 49's §2.3): the the membership's check (module 49's §3.3) — the the no stale's (module 49's §3.2):
  await requireOrgMember(session.userId, row.orgId)   // the module-49's line: the check is the membership's (module 49's §2.3) — the the 404's (module 50's)
  return row.orgId   // the module-49's line: the orgId is the session's fact (module 49's §1)
}
```

**The module-49's line:** the *active's is the implicit's* (module 49's §2.2) — the *the no active's is the null* (module 49's §3.2) — the *the check is the membership's* (module 49's §2.3) — the *the no stale's* (module 49's §3.2).

### 3.3 The membership's check (module 49's §2.3 — the `requireOrgMember`)

`FILE: src/lib/rbac.ts` (production pattern — [SERVER] — the module-49's §3.3: the `requireOrgMember`, the 404's)

```ts
// THE MEMBERSHIP'S CHECK (module 49's §2.3 — the requireOrgMember (module 49's §3.3) — the the 404's (module 50's)):
export async function requireOrgMember(userId: string, orgId: string) {
  const role = await getRole(userId, orgId)   // the module-48's §3.2 (module 48's §3.2)
  if (!role) throw new AppError({ status: 404, code: 'org.not_found', message: 'Organization not found' })   // the module-50's line: the 404 is the no leak (module 50's) — the the no 403's (module 50's)
}
```

**The module-49's line:** the *membership's check is the `requireOrgMember`* (module 49's §3.3) — the *the 404 is the no leak* (module 50's) — the *the no 403's* (module 50's).

## 4. Production Code — The cross-tenant test (the module-11-02's anchor)

### 4.1 The test's definition (module 49's §4.1 — the canonical)

`FILE: src/tests/cross-tenant.test.ts` (production pattern — the module-49's §4.1: the cross-tenant's test, the module-11-02's anchor)

```ts
// THE CROSS-TENANT'S TEST (module 49's §4.1 — the module-11-02's anchor — the the canonical (module 49's §4.1)):
import { describe, it, expect } from 'vitest'
import { setupOrgs } from '@/tests/fixtures'   // the the org A + the org B + the user A + the user B (module 49's §4.1)

describe('the cross-tenant (module 11-02)', () => {
  // THE 1: THE READ'S (module 49's §4.1): the the user B's read the org A's product → the 404 (module 50's) — the the no leak (module 50's):
  it('the user B cannot read the org A's product', async () => {
    const { orgA, userB, productA } = await setupOrgs()
    const result = await getProduct(userB.userId, orgA.id, productA.id)   // the module-17's service (module 17's) — the the orgId's scope (module 17's rule 1)
    expect(result).toBeNull()   // the module-49's line: the no member is the null (module 49's §4.1) — the the 404's (module 50's)
  })

  // THE 2: THE ACTION'S (module 29's): the the user A's markShipped the org B's order → the 404 (module 50's) — the the no leak (module 50's):
  it('the user A cannot markShipped the org B's order', async () => {
    const { orgB, userA, orderB } = await setupOrgs()
    await expect(transitionOrder(userA.orgId, orderB.id, 'shipped')).rejects.toMatchObject({ status: 404 })   // the module-40's §1.2 (module 40's) — the the orgId's scope (module 17's rule 1)
  })

  // THE 3: THE API KEY'S (module 36's): the the org A's key + the org B's URL → the 403 (module 50's) — the the mismatch (module 36's §4):
  it('the org A's key cannot access the org B's URL', async () => {
    const { orgA, orgB, keyA } = await setupOrgs()
    const result = await verifyApiKey(keyA, orgB.id)   // the module-36's §4 (module 36's) — the the key's org = the URL's org (module 36's §4)
    expect(result).toBeNull()   // the module-49's line: the mismatch is the null (module 49's §4.1) — the the 403's (module 50's)
  })

  // THE 4: THE UNIQUENESS'S (module 38's §1): the the org A's SKU = the org B's SKU → the success (module 49's §4.1) — the the per-tenant's (module 38's §1):
  it('the org A can use the org B's SKU', async () => {
    const { orgA, orgB } = await setupOrgs()
    await createProduct(orgB.id, { sku: 'SHARED' })   // the module-38's §1 (module 38's) — the the (org, sku) unique (module 38's §1)
    await expect(createProduct(orgA.id, { sku: 'SHARED' })).resolves.toBeDefined()   // the module-49's line: the per-tenant's is the success (module 49's §4.1)
  })
})
```

**The module-49's line:** the *cross-tenant's test is the 4 cases* (module 49's §4.1) — the *the read's is the 404* (module 50's) — the *the action's is the 404* (module 50's) — the *the API key's is the 403* (module 50's) — the *the uniqueness's is the success* (module 38's §1).

### 4.2 The test's 4 cases (module 49's §4.2 — the table)

| Case | The input | The expected | The module-49's line |
|---|---|---|---|
| **The read's** (module 49's §4.1) | the user B's read the org A's product | the 404 (module 50's) | the *the no member is the null* (module 49's §4.1) — the *the no leak* (module 50's) |
| **The action's** (module 29's) | the user A's markShipped the org B's order | the 404 (module 50's) | the *the no member is the 404* (module 49's §4.1) — the *the no leak* (module 50's) |
| **The API key's** (module 36's) | the org A's key + the org B's URL | the 403 (module 50's) | the *the mismatch is the 403* (module 36's §4) — the *the key's org = the URL's org* (module 36's §4) |
| **The uniqueness's** (module 38's §1) | the org A's SKU = the org B's SKU | the success (module 49's §4.1) | the *the per-tenant's is the success* (module 38's §1) — the *the (org, sku) unique* (module 38's §1) |

## 5. Common Mistakes (the tenancy failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The caller's orgId** (module 49's §1's line violated) | the *module-49's line: the orgId is the session's* (module 49's §1) — the *the caller's orgId is the *cross-tenant* (module 11-02's) — the *module-49's line: the orgId is the session's* (module 49's §1) — the *no caller's orgId* (module 49's §1)* | the *the session's orgId* (module 49's §1) — the *module-49's line: the orgId is the session's* (module 49's §1)* |
| **The no membership's check** (module 49's §2.3's line violated) | the *module-49's line: the check is the membership's* (module 49's §2.3) — the *the no check is the *cross-tenant* (module 11-02's) — the *module-49's line: the check is the membership's* (module 49's §2.3) — the *no cross-tenant* (module 49's §2.3)* | the *the `requireOrgMember`* (module 49's §3.3) — the *module-49's line: the check is the membership's* (module 49's §2.3)* |
| **The 403's leak** (module 50's line violated: the no 404) | the *module-50's line: the 404 is the no leak* (module 50's) — the *the 403's leak is the *existence* (module 50's) — the *module-49's line: the 404 is the no leak* (module 50's) — the *no 403's leak* (module 50's)* | the *the 404* (module 50's) — the *module-50's line: the 404 is the no leak* (module 50's)* |
| **The no per-tenant's unique** (module 38's §1's line violated) | the *module-38's line: the (org, sku) unique* (module 38's §1) — the *the no per-tenant's is the *cross-org's block* (module 38's §1) — the *module-49's line: the per-tenant's is the success* (module 49's §4.1) — the *no cross-org's block* (module 38's §1)* | the *the (org, sku) unique* (module 38's §1) — the *module-38's line: the (org, sku) unique* (module 38's §1)* |
| **The no cache's orgId** (module 25's line violated) | the *module-25's line: the orgId is the cache's key* (module 25's) — the *the no orgId is the *cache's cross-tenant* (module 25's) — the *module-49's line: the orgId is the cache's key* (module 25's) — the *no cache's cross-tenant* (module 25's)* | the *the orgId in the key* (module 25's) — the *module-25's line: the orgId is the cache's key* (module 25's)* |
| **The API key's no check** (module 36's §4's line violated) | the *module-36's line: the key's org = the URL's org* (module 36's §4) — the *the no check is the *cross-tenant* (module 36's §4) — the *module-49's line: the key's org = the URL's org* (module 36's §4) — the *no cross-tenant* (module 36's §4)* | the *the `keyOrgId === urlOrgId`* (module 36's §4) — the *module-36's line: the key's org = the URL's org* (module 36's §4)* |
| **The no cross-tenant's test** (module 49's §4's line violated) | the *module-49's line: the cross-tenant's test is the 4 cases* (module 49's §4.1) — the *the no test is the *no proof* (module 49's §4.1) — the *module-49's line: the cross-tenant's test is the 4 cases* (module 49's §4.1) — the *no proof* (module 49's §4.1)* | the *the `cross-tenant.test.ts`* (module 49's §4.1) — the *module-49's line: the cross-tenant's test is the 4 cases* (module 49's §4.1)* |

## 6. Security Notes

- **The orgId is the session's** (module 49's §1): the *module-49's line: the orgId is the session's* (module 49's §1) — the *module-19-02's cross-tenant* (module 75's) — the *the no caller's orgId* (module 49's §1).
- **The 404 is the no leak** (module 50's): the *module-50's line: the 404 is the no leak* (module 50's) — the *module-19-02's IDOR* (module 75's) — the *the no 403's leak* (module 50's).
- **The per-tenant's unique** (module 38's §1): the *module-38's line: the (org, sku) unique* (module 38's §1) — the *module-49's line: the per-tenant's is the success* (module 49's §4.1) — the *the no cross-org's block* (module 38's §1).
- **The cache's orgId** (module 25's): the *module-25's line: the orgId is the cache's key* (module 25's) — the *module-49's line: the orgId is the cache's key* (module 25's) — the *module-25's* *deep-dive* (module 25's).
- **The API key's org** (module 36's §4): the *module-36's line: the key's org = the URL's org* (module 36's §4) — the *module-49's line: the key's org = the URL's org* (module 36's §4) — the *module-36's* *deep-dive* (module 36's).
- **The least-privilege** (module 37's §6): the *module-37's line: the app's user is the least* (module 37's §6) — the *module-49's line: the member is the read-only* (module 48's §3.1) — the *module-37's* *deep-dive* (module 37's).

## 7. Performance Notes

- **The org's resolution is the per-request** (module 49's §7.1): the *module-49's line: the org's resolution is the per-request* (module 49's §7.1) — the *module-20's* *the cache's* (module 20's) — the *module-20's* *deep-dive* (module 20's).
- **The `use cache: private` is the session's** (module 47's): the *module-47's line: the `use cache: private` is the session's* (module 47's) — the *module-49's line: the org's read is the session's read* (module 47's) — the *module-47's* *deep-dive* (module 47's).
- **The N+1's org** (module 39's): the *module-39's line: the N+1 is the topology's* (module 39's) — the *module-49's line: the org's N+1 is the list's* (module 39's) — the *module-39's* *deep-dive* (module 39's).

## 8. Exercise

**Beginner.** *The org's resolution* (module 49's §2–3): the *the `getOrgFromRoute`* (module 3.1's) + the *the `getActiveOrg`* (module 3.2's) + the *the `requireOrgMember`* (module 3.3's) — *build it* — the *artifact: the 3 functions* (module 20's).

**Intermediate.** *The cross-tenant's test* (module 49's §4): the *the 4 cases* (module 4.1's) — the *the read's* + the *the action's* + the *the API key's* + the *the uniqueness's* — *build it* — the *artifact: the 4 cases' outputs* (module 20's).

**Production.** *The cache's orgId* (module 25's): the *the orgId in the key* (module 25's) + the *the cross-tenant's cache's test* (module 25's) — the *artifact: the cache's test's output* (module 20's).

## 9. Architecture Challenge

**Prompt:** The *"the team wants to add a 'personal' org: the no member's, the only the user's"* (the *module-49's* *org's* — the *module-48's* *role's* — the *module-49's line: the tenancy is the org's* (module 49's §1) — the *module-48's line: the role is the label* (module 48's §1.1) — the *module-49's standing line: the tenancy is the org's + the role is the label* (module 49's §1 + module 48's §1.1)).

The *problems*: (1) the *the personal's org* (the *the `personal`* (module 49's §9) — the *module-49's line: the personal's is the org's* (module 49's §9) — the *module-49's standing line: the personal's is the org's* (module 49's §9)).

(2) the *the owner's role* (the *the `owner`* (module 48's §3.1) — the *module-48's line: the owner is the all* (module 48's §3.1) — the *module-49's standing line: the owner is the all* (module 48's §3.1)).

**Design**: the *the addition* (the *the `personal` org* (module 49's §9) + the *the `owner` role* (module 48's §3.1) + the *the no re-learn* (module 49's §1) — the *module-49's line: the tenancy is the org's + the role is the label* (module 49's §1 + module 48's §1.1) — the *module-49's standing line: the personal's is the org's + the owner is the all + the no re-learn* (module 49's §9 + module 48's §3.1 + module 49's §1)).

Produce: the *the addition* (the *the `personal` org* (module 49's §9) + the *the `owner` role* (module 48's §3.1) + the *the no re-learn* (module 49's §1) — the *module-49's line: the tenancy is the org's + the role is the label* (module 49's §1 + module 48's §1.1) — the *module-49's standing line: the personal's is the org's + the owner is the all + the no re-learn* (module 49's §9 + module 48's §3.1 + module 49's §1)).

<details>
<summary>Model answer</summary>
**The addition** (module 49's §9 + module 48's §3.1):
1. **The personal's org** (module 49's §9): the *the `personal`* (module 49's §9) — the *module-49's line: the personal's is the org's* (module 49's §9).
2. **The owner's role** (module 48's §3.1): the *the `owner`* (module 48's §3.1) — the *module-48's line: the owner is the all* (module 48's §3.1).
**The generalization** (the *addition's* pattern, the *module's* standing rule): **the *personal's is the org's* (module 49's §9) — the *the owner is the all* (module 48's §3.1) — the *the no re-learn* (module 49's §1) — the *module-49's standing line: the personal's is the org's + the owner is the all + the no re-learn* (module 49's §9 + module 48's §3.1 + module 49's §1)*.
</details>

## 10. Official Documentation

- The OWASP Access Control Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Access_Control_Cheat_Sheet.html
- Better Auth: Organization plugin: https://www.better-auth.com/docs/plugins/organization
- The module-48's RBAC: the module-48 (the phase-11's file-01)
- The module-50's 401/403: the module-50 (the phase-11's file-03)

## 11. What You Should Know Before Continuing

- [ ] I can state the *rule* (module 1's: the orgId is the session's, never the caller's) — the *module-49's line: the orgId is the session's* (module 1's)
- [ ] I know the *3 lines* (module 1.1's: the service's + the query's + the DB's) — the *module-19's defense in depth* (module 19's)
- [ ] I know the *org's resolution* (module 2's: the route's + the active's + the check) — the *the `requireOrgMember` is the 404's* (module 3.3's)
- [ ] I know the *cross-tenant's test is the 4 cases* (module 4.1's) — the *the read's is the 404* + the *the action's is the 404* + the *the API key's is the 403* + the *the uniqueness's is the success* (module 4.1's)
- [ ] I know the *404 is the no leak* (module 50's) — the *the no 403's leak* (module 50's)
- [ ] I know the *per-tenant's unique* (module 38's §1) — the *the (org, sku) unique* (module 38's §1)
- [ ] I've done the *org's resolution* (module 8's beginner) + the *cross-tenant's test* (module 8's intermediate) + the *cache's orgId* (module 8's production) — the *artifacts* (module 20's)

**Next:** Module 50 — Frontend vs Server Boundaries; 401 vs 403 (the *the 401 is the who* — the *the 403 is the what* — the *module-50's line: the 401 is the who, the 403 is the what* (module 50's)).
