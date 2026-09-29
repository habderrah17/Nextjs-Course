# Module 13 — Client Components & the Boundary

**Phase 3: Server/Client Components · Module 13 of 101**

> **Where does this run?** `[CLIENT]` — this module is about the half that *does* ship. The skill: knowing the exact line where "server" stops.

---

## 1. Concept — `"use client"` is a boundary marker, not a runtime

The directive at the top of a file tells the compiler: *this file is a Client Component; everything it imports is reachable from the browser; bundle it; hydrate it.* Three precise consequences:

1. **The ratchet:** inside a client file, every import is client (recursively). You can import a *server* component's **file** only if that file is also client — a client file **cannot import a server component** (no directive = server). The boundary is one-directional: **server components render client components; never the reverse.**
2. **The bundle:** the file (and its client-only imports) becomes a JS chunk, loaded on routes that use it, and **hydrated**.
3. **The props channel:** a server component renders it with *serializable props only* (module 13-serialization). Functions can cross **one specific way**: a **Server Function reference** (`'use server'`) can be passed as a prop (or imported from a `'use server'` file) — it crosses as an opaque ID the client POSTs back to.

**What "hydration" does** (connect to your React knowledge): React takes the server-rendered DOM, attaches listeners, initializes state/effects, and *diffs against the client render* — if the client's first render differs from the server HTML, you get a **hydration mismatch** (dev: red overlay; prod: React patches the DOM, silently). Mismatches are a module-25 debugging topic; the rule that prevents 90% of them: **client components must render identically on first paint to what the server rendered** (no `Date.now()`, no `Math.random()`, no `window.innerWidth` in the initial render — use `useEffect` for divergence).

## 2. Mental Model — the island architecture

The app is a **sea of server rendering with islands of interactivity**:

```
┌─────────────────────────────────────────────────────┐
│ layout.tsx [SERVER]                                  │
│  ┌───────────────────────────────────────────────┐  │
│  │ page.tsx [SERVER]                             │  │
│  │   ┌─────────────┐  ┌────────────────────────┐ │  │
│  │   │ data section│  │ <Toolbar/> [CLIENT]    │ │  │
│  │   │ (DB → DTOs) │  │  ├ <SearchBox/> (state)│ │  │
│  │   └─────────────┘  │  └ <ExportButton/>     │ │  │
│  │                     │    (calls action)      │ │  │
│  │                     └────────────────────────┘ │  │
│  └───────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────┘
```

**The island contract** (what every client component in this course obeys):

1. Receives **data as props** (DTOs, already fetched by the server).
2. Receives **actions as props/imports** (Server Function references for mutations).
3. Owns **only interaction state** (open/closed, draft text, selection, animation).
4. Has **no data-fetching of its own** (except deliberately client-cached data — module 23-03's decision).
5. Is **as small as the interactivity requires** — a 200-line page where only the toolbar is interactive becomes a 200-line server page + a 40-line client toolbar.

**The 5-step decision** (from the mental-models file, applied here): browser API? UI state? event handler? client-only library (Motion, a charting lib, a drag lib)? If any → client. If none → server, no exceptions.

## 3. Architecture — the three ways actions reach the client

```mermaid
flowchart LR
    subgraph SERVER
        F["actions.ts<br/>'use server' (file-level)"]
        P["page.tsx [SERVER]<br/>inline 'use server' fn"]
    end
    subgraph CLIENT
        A["<form action={createProduct}>"]
        B["<button formAction={deleteItem}>"]
        C["onClick={() => updateName(id)}"]
        D["<RowActions onSave={saveAction}>"]
    end
    F -->|import (client may import 'use server' files)| A
    F --> B
    F --> C
    P -->|prop| D
```

1. **Import** — a client file imports from a file whose *top* is `'use server'`. The imported function is a reference, callable like a local async function (module 07-01 deep-dives invocation mechanics).
2. **Prop** — a server component passes the action into a client island: `<RowActions onSave={saveRow} />`. The prop type is `(formData: FormData) => Promise<void>` (or the typed variant).
3. **Form semantics** — `action`/`formAction`/`form` attributes wire the action to native forms (progressive enhancement, module 07-02).

All three arrive at the same place: a **POST** carrying a serialized argument → server runs the function → response carries the new RSC payload + any `redirect()`.

## 4. Production Code

### 4.1 The client island (orders table — the capstone's flagship island)

`FILE: src/features/order/components/orders-table.tsx` (production pattern — [CLIENT])

```tsx
'use client'

import { useState } from 'react'
import Link from 'next/link'
import { Table, TableBody, TableCell, TableHead, TableHeader, TableRow } from '@/components/ui/table'
import { OrderStatusBadge } from '@/components/order-status-badge'      // [BOTH] pure presentational — no directive (see note below)
import { cancelOrder } from '../order-actions'                            // 'use server' file — a Server Function reference
import type { OrderDTO } from '@/types/order'

// NOTE the import rule in action: a client file may import a directive-free file only if
// that file contains no server-only code — importing it compiles the file into the client
// bundle (module 03's 5 import rules). OrderStatusBadge is a pure presentational primitive
// (no hooks that need the server, no service imports), so it's safe as [BOTH]. (Convention:
// presentational primitives live in components/ with no directive; the moment one imports a
// service, it becomes server-only and the client import is a build error — module 09-01's guard.)

export function OrdersTable({ orders, nextCursor }: { orders: OrderDTO[]; nextCursor: string | null }) {
  const [cancellingId, setCancellingId] = useState<string | null>(null)

  async function handleCancel(order: OrderDTO) {
    setCancellingId(order.id)
    try {
      await cancelOrder(order.id)          // Server Function call — POST happens under the hood
      // The server action revalidated tags; the router updates the page automatically
      // (RSC re-render) — no manual refetch here.
    } finally {
      setCancellingId(null)
    }
  }

  return (
    <Table>
      <TableHeader>
        <TableRow>
          <TableHead>Order</TableHead><TableHead>Customer</TableHead>
          <TableHead>Status</TableHead><TableHead>Total</TableHead><TableHead aria-label="Actions" />
        </TableRow>
      </TableHeader>
      <TableBody>
        {orders.map((order) => (
          <TableRow key={order.id}>
            <TableCell><Link className="font-medium" href={`/orders/${order.id}`}>#{order.number}</Link></TableCell>
            <TableCell>{order.customerName}</TableCell>
            <TableCell><OrderStatusBadge status={order.status} /></TableCell>
            <TableCell>{order.totalFormatted}</TableCell>
            <TableCell className="text-right">
              {order.status === 'pending' && (
                <button
                  type="button"
                  onClick={() => handleCancel(order)}
                  disabled={cancellingId === order.id}
                  className="text-sm text-destructive disabled:opacity-50"
                >
                  {cancellingId === order.id ? 'Cancelling…' : 'Cancel'}
                </button>
              )}
            </TableCell>
          </TableRow>
        ))}
      </TableBody>
    </Table>
  )
  // Pagination: server-rendered <Pagination/> below the table (module 10).
}
```

**Why this shape is "good":** the table *renders* server data (props), *acts* via a Server Function, and its only state is `cancellingId`. Nothing here knows about the database, the session, or the cache.

### 4.2 A client component that needs browser APIs (theme toggle)

`FILE: src/components/theme-toggle.tsx` (production pattern — [CLIENT])

```tsx
'use client'

import { useEffect, useState } from 'react'
import { Moon, Sun } from 'lucide-react'

// Reads an initial value from a data attribute set server-side (no flash),
// then owns the browser-only concern: localStorage + the <html> class.
export function ThemeToggle() {
  const [mounted, setMounted] = useState(false)
  const [theme, setTheme] = useState<'light' | 'dark'>('light')

  useEffect(() => {
    setTheme((document.documentElement.classList.contains('dark') ? 'dark' : 'light'))
    setMounted(true)
  }, [])

  function toggle() {
    const next = theme === 'dark' ? 'light' : 'dark'
    setTheme(next)
    document.documentElement.classList.toggle('dark', next === 'dark')
    localStorage.setItem('theme', next)
  }

  if (!mounted) return <span className="inline-block h-9 w-9" aria-hidden />  // stable size → no CLS

  return (
    <button type="button" onClick={toggle} aria-label={`Switch to ${theme === 'dark' ? 'light' : 'dark'} mode`}
      className="rounded-md p-2 hover:bg-muted">
      {theme === 'dark' ? <Sun className="h-4 w-4" /> : <Moon className="h-4 w-4" />}
    </button>
  )
}
```

Note the hydration discipline: `mounted` gate + placeholder of identical size — the classic pattern for *any* client component whose first render depends on a browser value.

### 4.3 The BAD architecture (study this)

`FILE: src/app/(BAD)/dashboard/page.tsx` (BAD — demonstration only)

```tsx
'use client'

import { useEffect, useState } from 'react'

export default function DashboardPage() {
  const [orders, setOrders] = useState<OrderDTO[] | null>(null)
  const [error, setError] = useState<string | null>(null)

  useEffect(() => {
    // The page is client, so the server boundary is GONE:
    fetch('/api/orders')                    // → own Route Handler → DB (the hop chain)
      .then((r) => r.json())
      .then(setOrders)
      .catch(() => setError('Failed to load'))
  }, [])

  if (error) return <ErrorState message={error} />
  if (!orders) return <DashboardSkeleton />
  return <OrdersClient orders={orders} />   // more client code, fetching again inside
}
```

**What broke (the checklist):** client JS for the *entire* page shipped; zero HTML for crawlers/first paint (LCP = data render); a redundant API hop (server→own route→DB); two auth checks instead of one; no progressive enhancement; no streaming (one `useEffect` = one waterfall); and — the quiet killer — **the page can no longer be cached at the edge** (it's a client route).

**The refactor** is exactly §4 of module 12 (server page + islands). Every module from here assumes the refactored shape.

## 5. Common Mistakes

| Mistake | Fix |
|---|---|
| `"use client"` "to be safe" on everything | The island contract (§2); default server |
| Client component importing a *server-only* module (service, db) | Build error if `server-only` guard present — keep it |
| `window` access during render (not in an effect) | Crashes server render or hydration mismatch; gate with `useEffect`/`mounted` |
| Passing `new Date()` as a prop (server formats one way, client another) | Serialize as ISO string; format in ONE place |
| `Math.random()`/`Date.now()` in a client component's *initial* render | Hydration mismatch; move to `useEffect` or accept server-generated values |
| Forgetting a client component must be *rendered by* a server component (or another client) — you can't "import" a server component into it | Restructure: the server parent renders both |

## 6. Security Notes

- The client bundle contains **all** client components and their imports: no secrets, no service code, no DB URLs — the build enforces it (module 09-01), but *review* still catches the "clever" ones (a `String(process.env.X)` that compiles into the bundle).
- Server Function props are **attacker-controllable**: a client reference can be invoked with *any* serializable argument by a crafted POST — validation inside the action is non-negotiable (module 07-03).
- What the client *renders* is public: never conditionally hide sensitive data in client state expecting it to be "invisible."

## 7. Performance Notes

- Client component count ≈ hydration cost + bundle size. The budget habit (module 18-03): per route, list the client islands; each must name the interactivity it owns.
- Islands should be **route-scoped**: an island used on one route shouldn't be in the initial chunk of another (verify with the bundle analyzer).
- The `mounted`-gate pattern (§4.2) costs nothing and is your CLS/hydration insurance.

## 8. Exercise

**Beginner.** Convert the §4.3 BAD page into the refactored shape with stub data. Count: client islands (should be ≤ 2), `fetch` calls (0 in the client), and the HTML present before JS (the whole page). Write the before/after in `docs/island-refactor.md`.

**Intermediate.** Build a `SearchBox` client component (draft state + debounce + submit writes to the URL) that a *server* page renders with `initialQuery` as a prop. Prove hydration identity: force a mismatch by reading `window.innerWidth` in the first render, observe the dev overlay, then fix it with the `mounted` gate. Explain the overlay's diff.

**Production.** Audit your capstone: for every client component, write (a) the interactivity it owns, (b) the props it receives, (c) the actions it invokes. Flag any component failing the island contract. This table is the artifact a senior reviewer would ask for.

## 9. Architecture Challenge

**Prompt:** The orders table needs four behaviors: (1) cancel order (mutation), (2) row-click expands inline details (fetches full order), (3) CSV export of current filtered view, (4) "auto-refresh every 30s" for live status.

For each: client island or server, and *how* the data flows (props? action? client fetch? interval?). Which one is the **only** legitimate client fetch, and why?

<details>
<summary>Model answer</summary>
(1) Cancel: client island button → Server Function (`cancelOrder`) → `updateTag('orders')` → RSC re-render updates the row. No client data flow beyond the call.
(2) Expand: the details are *more of the same data* the server already has — the server page fetches the full order for the expanded row… but expansion is client state (which row is open). Resolution: the server renders **collapsed** rows with enough data to collapse; the *expanded* view is a small Suspense hole — a server component `<OrderDetails id>` inside a client wrapper that mounts/unmounts on expand, wrapped so its fetch streams. (Simpler production variant: include details in the row DTO for the first 10 rows — data you know the user will expand — and stream deeper rows on demand.)
(3) CSV export of the *current filtered view*: the filter is URL state → the export is a **Route Handler** `GET /orders/export?…filters` (module 08) — the browser navigates/downloads; the server re-runs the same query. Not a client fetch: it's a *resource* (a file), the canonical Route Handler case.
(4) Auto-refresh: the only legitimate client fetch — and still, prefer the framework: a client island calling `router.refresh()` on an interval (or `useEffect` + `refresh`) re-requests the RSC data with the *same* caching/invalidation semantics as everything else. A raw `fetch('/api/orders')` + manual state would fork the data path — forbidden by the island contract. (If 30s polling of a heavy page is needed, the real answer is to shorten that section's `cacheLife` or use SSE — module 18/22 — not more client code.)
</details>

## 10. Official Documentation

- Server and Client Components: https://nextjs.org/docs/app/getting-started/server-and-client-components
- React — "use client": https://react.dev/reference/rsc/use-client
- React — RSC serialization rules: https://react.dev/reference/rsc/use-client#serializable-types
- Hydration errors: https://nextjs.org/docs/messages/hydration-error
- `"use server"`: https://react.dev/reference/rsc/use-server

## 11. What You Should Know Before Continuing

- [ ] I can state the three consequences of `"use client"` (ratchet, bundle, props channel)
- [ ] I can recite the island contract (5 rules) and audit a component against it
- [ ] I know the three ways actions reach the client (import, prop, form semantics)
- [ ] I can explain hydration mismatch and the `mounted`-gate fix
- [ ] I can name the ONE legitimate client-fetch case in the challenge (interval refresh via `router.refresh()`)

**Next:** Module 14 — Serialization: the exact rules of what crosses the boundary.
