# Module 03 — Project Structure: Folders That Earn Their Place

**Phase 1: Foundations · Module 3 of 101**

> **Where does this run?** Structure decisions shape *both* sides: which files are allowed to exist server-side only, and which get bundled to the client. Folder layout is a **boundary tool**, not decoration.

---

## 1. Concept — Why the structure exists (and what it does to the build)

Three forces shape a Next.js project:

1. **The framework's file conventions are non-negotiable.** Files in `app/` (or `src/app/`) have framework meaning (`page.tsx` = a route, `proxy.ts` = request interception, `route.ts` = HTTP endpoint). You cannot move `app/` somewhere the framework can't find it.
2. **The bundler decides what ships.** Anything reachable by import from a `"use client"` file ends up in a client chunk. Folder structure is how you keep server-only code *out of reach* of client code — enforced by convention, and by `server-only` guards (module 09-01).
3. **Humans decide what they can find.** At 30 files, "layers" work. At 300, only **features** (domain boundaries) stay navigable.

The classic argument — layer-based vs feature-based:

```
Layer-based (classic):            Feature-based (this course):
components/                        features/
  button/                            product/
  table/                               components/   (UI for products)
  dialog/                                actions.ts ('use server')
services/                            orders/
  product-service.ts                   user/
  order-service.ts                     admin/
actions/                           components/        ← only truly shared UI
  product-actions.ts                   lib/ services/ db/ schemas/
```

**Decision: hybrid.** Framework-mandated directories (`app/`) are organized *by route* (they must be). Everything else is organized *by feature*, with a small number of **shared layers** that features import. A feature owns its UI, its actions, its services, and its schemas. Shared code moves to the shared layers *when a second feature needs it* — never preemptively (abstractions are born from duplication, not prophecy).

## 2. Mental Model — the boundary map of `src/`

```
src/
├── app/                    [MIXED — the route tree]
│   ├── (marketing)/        ← public, SEO-critical, mostly [SERVER]
│   ├── (app)/              ← authenticated user area
│   ├── (admin)/            ← admin console (route group, distinct layout)
│   └── api/                [SERVER] Route Handlers (public API, webhooks, export)
├── features/               [MIXED — domain features; each owns its boundary]
│   ├── product/
│   │   ├── components/     (client islands: filters, table, dialogs)
│   │   ├── product-actions.ts   ('use server')
│   │   └── product-schemas.ts   (Zod: form + input schemas)
│   ├── order/  ├── user/  ├── cart/  ├── admin-user/  ├── organization/
├── components/             [SHARED UI — no feature knowledge]
│   ├── ui/                 (shadcn-generated primitives)
│   ├── button.tsx, table.tsx, skeleton.tsx, empty-state.tsx, ...
├── services/               [SERVER ONLY — the only code that touches the ORM]
│   ├── products.ts  orders.ts  users.ts  organizations.ts  analytics.ts
├── db/                     [SERVER ONLY — schema, client, migrations]
│   ├── schema.ts  index.ts (drizzle client + `import 'server-only'`)
├── actions/                [SERVER ONLY — cross-feature 'use server' (small; most actions live in features)]
├── schemas/                [BOTH — Zod schemas shared client+server (forms, env parsing)]
│   ├── env.ts  z-ids.ts (uuid/email helpers)
├── hooks/                  [CLIENT ONLY — useDebounce, useMediaQuery, useSession wrapper]
├── lib/                    [MIXED — small utilities; each file labeled]
│   ├── auth.ts [SERVER]  utils.ts [BOTH]  format.ts [BOTH]
├── types/                  [BOTH — DTO types shared across the boundary]
├── config/                 [BOTH — site config (name, URL, nav links) — no secrets]
├── proxy.ts                [SERVER — request interception]
└── middleware-free note: in 16.x this file is proxy.ts (Node runtime)
```

**Import rules (enforced in code review + CI audit from module 02):**

1. `services/`, `db/`, `app/api/` may be imported **only** from `[SERVER]` files.
2. `hooks/` and `features/*/components` that are client must never import `services/` or `db/`.
3. `schemas/` and `types/` are the **only** cross-boundary value modules — they contain no imports from `lib/auth`, `db`, or `services`.
4. Features import shared layers; shared layers never import features.
5. A file that needs to cross the boundary is a *signal to restructure* — create a DTO, an action, or a client island. Don't make the boundary permeable.

## 3. Architecture — how the structure serves the five surfaces

```mermaid
flowchart TD
    subgraph CLIENT_FILES["[CLIENT] — bundled"]
        HF[features/*/components (use client)]
        HK[hooks/]
        SHARED[components/]
    end
    subgraph SERVER_FILES["[SERVER] — never shipped"]
        SVC[services/] --> DB[db/]
        FA[features/*/product-actions.ts<br/>'use server'] --> SVC
        API[app/api/* route.ts] --> SVC
        AUTH[lib/auth.ts] --> DB
        PR[proxy.ts]
    end
    subgraph BOTH["[BOTH] — pure data"]
        SC[schemas/]
        TY[types/]
        CFG[config/]
    end
    HF -->|props/DTOs| TY
    HF --> SC
    SHARED --> SC
    FA -.->|action ref crosses| HF
```

## 4. Production Code — the capstone tree at Stage 1 (and how it grows)

`FILE: (tree)` — target after Phase 1:

```
commerce-ops/
└── src/
    ├── app/
    │   ├── layout.tsx
    │   ├── page.tsx
    │   ├── globals.css
    │   └── (marketing)/            ← added in Phase 2
    │       └── layout.tsx
    ├── components/
    │   └── ui/                     ← filled in Phase 13 (shadcn)
    ├── features/
    ├── schemas/
    │   └── env.ts                  ← added in Phase 19, but created now (good habit)
    ├── lib/
    │   └── utils.ts
    ├── services/
    ├── db/
    ├── hooks/
    ├── types/
    └── config/
        └── site.ts
```

`FILE: src/config/site.ts` (production pattern — [BOTH])

```ts
// Site-wide, non-secret configuration. Referenced by metadata, nav, and marketing copy.
// NO env access here — config/ is a client-safe value module.
export const site = {
  name: 'Commerce Ops',
  shortDescription: 'Multi-tenant commerce & operations platform',
  // NEXT_PUBLIC_ access belongs in schemas/env.ts, not here.
} as const

export const nav = {
  public: [
    { title: 'Pricing', href: '/pricing' },
    { title: 'Blog', href: '/blog' },
    { title: 'Catalog', href: '/products' },
  ],
  user: [
    { title: 'Dashboard', href: '/dashboard' },
    { title: 'Orders', href: '/orders' },
    { title: 'Products', href: '/products/manage' },
    { title: 'Settings', href: '/settings' },
  ],
} as const
```

`FILE: src/lib/utils.ts` (production pattern — [BOTH])

```ts
/** Re-exported at the path shadcn/ui expects (`@/lib/utils`). */
import { clsx, type ClassValue } from 'clsx'
import { twMerge } from 'tailwind-merge'

export function cn(...inputs: ClassValue[]) {
  return twMerge(clsx(inputs))
}

export function slugify(input: string): string {
  return input.toLowerCase().normalize('NFKD').replace(/[^a-z0-9]+/g, '-').replace(/(^-|-$)+/g, '')
}
```

The growth schedule (each folder's *first citizen* is what triggers creation):

| Phase | New folder(s) | First citizen |
|---|---|---|
| 2 Routing | `app/(marketing)`, `app/(app)`, `app/(admin)` | marketing layout, dashboard layout |
| 4 Data | `services/`, `db/` | `services/products.ts`, `db/schema.ts` |
| 7 Actions | `features/product/{components,product-actions.ts}` | `createProduct` |
| 8 Route Handlers | `app/api/v1/...` | `GET /api/v1/products` |
| 10 Auth | `lib/auth.ts`, `app/api/auth/[...all]/route.ts` | Better Auth server+client |
| 12 Forms | `features/*/…-schemas.ts` | `product-form-schema` |
| 13 UI | `components/ui/`, `hooks/` | shadcn primitives, `use-debounce.ts` |
| 19 Security | `schemas/env.ts` (formalized), proxy headers | env validation, security headers |

## 5. Common Mistakes

| Mistake | Why it fails | Fix |
|---|---|---|
| `components/` becomes a god-folder (150 files) | No feature context; imports cross every domain | Split into `features/*/components`; keep only cross-feature primitives in `components/` |
| `utils.ts` at every level, 40 files | Duplication you can't search | One shared `lib/utils.ts`; feature-specific helpers stay in the feature |
| `services/product.ts` imported by a client component | The bundler drags the DB client into the client build (or crashes on `server-only`) | Services are server-only; client gets DTOs via props or a Route Handler (module 04-02, 08) |
| Pre-creating 12 empty folders "for the enterprise" | Empty folders die; they promise a structure the app never had | Create a folder with its first file (rule above) |
| `types/` importing from `lib/` | Type-only today, value-tomorrow; breaks the [BOTH] purity | DTO types are standalone or derived from Zod in `schemas/` |
| Putting `app/` inside `src/features/` | Framework can't find routes | `app/` location is fixed by convention |

## 6. Security Notes

- Structure is your **first** server-only enforcement: `db/` and `services/` are physically unreachable from client imports *by convention* — the `server-only` package (module 09-01) makes the convention a **build error**.
- `schemas/env.ts` will contain the runtime env *parser* — it must never be imported into client code; env parsing happens once at server boot (module 19-03).

## 7. Performance Notes

- Folder structure shapes **code splitting boundaries**: features that bundle their own client islands keep chunks route-scoped. A giant shared `components/` can bloat the initial chunk.
- Keep `components/ui/` lean — it is on every route that uses it.

## 8. Exercise

**Beginner.** Create the Stage-1 tree in your scaffold. For each of the 9 shared directories, write a one-line comment in a `docs/structure.md` file explaining *which force* (framework / bundler / humans) created it.

**Intermediate.** Add a deliberate violation: a client component in `features/product/components/` that imports `services/products.ts`. Run the build and the module-02 boundary audit. Record the exact failure. Then fix it the *right* way (service returns DTOs; server component passes data as props) and re-run.

**Production.** Write a `docs/structure-decisions.md` ADR (1 page): the layer-vs-feature decision, the 5 import rules, and the "folders are born with their first citizen" rule. Have a peer review it against your actual tree.

## 9. Architecture Challenge

**Prompt:** A 2-person team ships the capstone's admin area. Six months later, a 12-person team joins and needs to add "invoices," "payments," and "shipping" features. Your structure is the Stage-1 tree.

1. Which folders will be in pain first, and why?
2. What do you add *before* the first invoice feature lands (and what do you explicitly *not* add)?
3. Where would the invoice feature's Server Action live, and why not in `actions/`?

<details>
<summary>Model answer</summary>
1. `services/` (flat file-per-domain gets crowded, no naming convention), `app/(admin)` (route explosion — needs subgroups like `(admin)/invoices`), and the absence of a feature layer for domains that weren't features yet. `features/` will be created in a rush with inconsistent internal shapes.
2. Add: the `features/` convention *with a written template* (components/, `<domain>-actions.ts`, `<domain>-schemas.ts`), a `services/` naming convention (`<domain>.ts` + `list/get/create/update` signature standard), route-group plan for `(admin)`. Not added: a micro-frontend boundary, a "core" package, a monorepo, an abstraction layer over services — none of these are justified by three new domains; they are justified by deployment independence, which you don't have.
3. In `features/invoice/invoice-actions.ts` — colocation means the action, its schemas, and its UI travel together; cross-feature actions in `actions/` are reserved for genuinely cross-domain operations (e.g., a "suspend organization" that touches users + products + orders).
</details>

## 10. Official Documentation

- Project structure: https://nextjs.org/docs/app/getting-started/project-structure
- File conventions: https://nextjs.org/docs/app/api-reference/file-conventions
- `src` folder: https://nextjs.org/docs/app/api-reference/file-conventions/src-folder

## 11. What You Should Know Before Continuing

- [ ] I can state the three forces that shape structure and the five import rules
- [ ] I can draw the boundary map of `src/` and name which surfaces live where
- [ ] My scaffold has the Stage-1 tree, and I know what triggers each future folder
- [ ] I understand that folder layout is a server-only enforcement tool, with `server-only` as the build-time backstop

**Next:** Module 04 — TypeScript for Next.js: the exact language features professional Next.js code requires.
