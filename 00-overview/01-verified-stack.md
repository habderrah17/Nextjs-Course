# 01 — Verified Modern Next.js Stack

**Verified: September 28, 2026.** Methodology: official documentation pages and official release channels first (nextjs.org/docs, react.dev, package changelogs); third-party sources only for cross-checking, never as the source of an API fact. Where the official docs are explicit about a change, the course teaches the new behavior and explains the old one as legacy.

> **Current Next.js version used in this course: 16.3.6**
> (confirmed on https://nextjs.org/docs on 2026-09-28; 16.x GA was October 21, 2025; 15.x support ends October 2026).

Every major technology lists: **version · official docs · the important APIs this course teaches.**

---

## Framework core

### Next.js 16.3.6
- **Docs:** https://nextjs.org/docs (index: https://nextjs.org/docs/llms.txt)
- **Key APIs taught:**
  - App Router file conventions: `layout.tsx`, `page.tsx`, `loading.tsx`, `error.tsx`, `not-found.tsx`, `template.tsx`, `default.js` (parallel routes), route groups, dynamic `[param]`, catch-all `[...slug]`, optional catch-all
  - `Link` + prefetching; programmatic navigation; modern navigation architecture (layout deduplication, incremental/partial prefetching)
  - Server Components / Client Components; `"use client"` boundary; serialization rules
  - **Server Functions / Server Actions**: `'use server'` directive, `formAction`, progressive enhancement, sequential dispatch
  - **Route Handlers**: `GET/POST/PUT/PATCH/DELETE` in `route.ts`
  - **Proxy**: `proxy.ts` (formerly `middleware.ts`), `matcher` config, Node.js runtime
  - **Cache Components**: `cacheComponents: true`, `'use cache'` (data-level & UI-level), `cacheLife(profile)`, `cacheTag(tag)`, `revalidateTag(tag, profile)`, `updateTag(tag)`, `revalidatePath(path)`, `'use cache: remote'`, `'use cache: private'`
  - Async request APIs: `params`, `searchParams`, `cookies()`, `headers()`, `draftMode()` — **all awaited**
  - `generateMetadata`, `generateStaticParams`, `next/font`, `next/image`
  - `redirect()`, `notFound()`, `permanentRedirect()` in navigation
  - `next.config.ts` (Turbopack config, cache handlers, runtime config)
  - Turbopack (default bundler, file-system caching beta)
  - `next build` output & dev logs (Compile/Render breakdown)

### React 19.2
- **Docs:** https://react.dev
- **Key concepts taught:**
  - React Server Components (stable) — what renders where
  - Actions — `useActionState`, `useOptimistic`, `useFormStatus` (the React hooks behind Server Actions)
  - `Suspense` — streaming, nested boundaries
  - `startTransition` / transitions — how navigation and action dispatch interact with UI
  - React 19.2 additions: `useEffectEvent`, View Transitions API, `<Activity>` — where Next.js 16 surfaces them
  - `ref` as a prop (no more `forwardRef` boilerplate)

### React Compiler 1.0
- **Docs:** https://react.dev/learn/react-compiler/introduction-to-react-compiler
- **Status:** stable since October 2025; **stable support in Next.js 16**; a `create-next-app` prompt option; compiler-powered lint rules in `eslint-plugin-react-hooks`
- **Taught:** how automatic memoization works, what it replaces (`useMemo`/`useCallback`), the rules of thumb for code that the compiler can't handle, and how to enable it via `next.config.ts` / CLI.

---

## Language & tooling

### TypeScript 5.x (minimum 5.1)
- **Docs:** https://www.typescriptlang.org/docs/
- **Taught (as needed):** type-safe props, generics (service layer, hooks), discriminated unions (API/error types), utility types (`Partial`, `Pick`, `Record`), DTO design, env var typing, typed route params/searchParams, typed forms (Zod inference).

### Node.js 24 LTS ("Krypton")
- **Docs:** https://nodejs.org/docs/latest-v24.x/
- **Facts:** Next.js 16 requires **Node ≥ 20.9**; the course targets **24 LTS** (support to April 2028). Node 26 is "Current" (LTS transition October 2026) — not the course target.

### ESLint 9 (flat config) or Biome
- **Docs:** https://eslint.org/docs/latest/ · https://biomejs.dev/
- **Facts:** `next lint` was **removed in Next.js 16** — you run `eslint` (or `biome`) directly. `create-next-app` prompts ESLint / Biome / None. The course uses ESLint 9 flat config (broader ecosystem, includes `eslint-plugin-react-hooks` with React Compiler rules) and shows the Biome alternative.

---

## Data & backend

### PostgreSQL 18
- **Docs:** https://www.postgresql.org/docs/current/
- **Taught:** schema design (PK/FK, constraints, indexes, generated columns), query patterns, transactions, `EXPLAIN ANALYZE`, connection pooling (PgBouncer / managed poolers), multi-tenancy patterns (shared-schema scoping), JSONB where appropriate.

### Drizzle ORM v1 (primary ORM)
- **Docs:** https://orm.drizzle.team
- **Key APIs taught:** `pgTable`, `pgEnum`, relations, `select/insert/update/delete`, `db.transaction`, `and/eq/inArray/like/ilike`, `.limit/.offset` & cursor patterns, `drizzle-kit generate/migrate/push`, schema push vs migrations, `server-only` guard, custom types (`$type`), raw SQL escape hatch, connection config (`'connection_limit'`, pooling flags).

### Prisma 7 (alternative)
- **Docs:** https://www.prisma.io/docs
- **Taught in `09-database/06-prisma-alternative.md`:** schema language, `prisma migrate`, client queries, transactions, Accelerate/pooling, and the exact tradeoffs vs Drizzle.

### Zod 4
- **Docs:** https://zod.dev
- **Key APIs taught:** `z.object/string/number/enum/literal`, `.parse` vs `.safeParse`, `z.infer`, `z.input`/`z.output`, discriminated unions (`z.discriminatedUnion`), `z.email()` (v4 top-level), `z.coerce`, ZodEffects (`transform`), error mapping (`error` parameter, field errors), `z.record`, env schema parsing, **runtime validation at every trust boundary** (forms, Server Function inputs, env, external API responses).

### Better Auth 1.6.x (primary auth)
- **Docs:** https://better-auth.com/docs
- **Key APIs taught:** `betterAuth()` server config, Postgres (Kysely/Drizzle) adapter, `auth.api.signInEmail` / `signUpEmail` / `signOut`, `auth.api.getSession({ headers })`, `createAuthClient()` + `useSession()` React hook, session config (expiration, update age, cookie options), plugins: organization (multi-tenancy), MFA (TOTP), passkeys, email OTP/verification, rate limiting (built-in), password policies. **Schema ownership:** users/sessions/accounts/verifications tables generated in *your* database.
- **Context (verified 2026):** Auth.js (NextAuth) — v5 never reached stable; maintenance of the project transferred to the Better Auth team (Sept 2025); Vercel acquired Better Auth (July 2026); maintainers direct new projects to Better Auth. A 2026 critical fail-open CVE affected Auth.js — taught as a case study in "why the session store matters."

### Auth.js (legacy knowledge)
- **Docs:** https://authjs.dev
- **Taught as:** historical architecture (adapter pattern, JWT vs database sessions), migration awareness, and why its security model changed hands. Not the course's primary.

---

## Forms & client state

### React Hook Form 7.x
- **Docs:** https://react-hook-form.com
- **Key APIs taught:** `useForm`, `register`, `formState` (errors, isSubmitting, isDirty), `Resolver` integration with Zod (`@hookform/resolvers/zod`), `reset` with server errors, `mode` (onBlur/onChange), `watch`, server-action round-trip wiring.

### TanStack Query 5.x (evaluated, not assumed)
- **Docs:** https://tanstack.com/query/latest/docs/overview
- **Taught:** when Next.js server-side data + Cache Components is *enough*, and when a client cache earns its cost: polling, client-driven mutations with background updates, offline, highly interactive client data. Includes the official "client-side data fetching" guide pattern (initial data from a Server Component into the Query cache).

### Zustand 5.x (chosen global client state)
- **Docs:** https://zustand.docs.pmnd.rs
- **Taught:** the decision framework (local → URL → server → form → global) and a real example (UI shell state). Redux Toolkit covered in the tradeoff module.

---

## UI & styling

### Tailwind CSS 4.x
- **Docs:** https://tailwindcss.com/docs
- **Key APIs taught:** CSS-first configuration (`@import "tailwindcss"`, `@theme`, `@custom-variant`), OKLCH color system, `@source` auto-detection, container queries, dark mode via class strategy, design tokens as CSS variables, cascade layers.

### shadcn/ui (current, Tailwind v4 + React 19)
- **Docs:** https://ui.shadcn.com/docs
- **Architecture taught:** the **registry + CLI copy model** (you own the source), `npx shadcn@latest init/add`, `components.json`, OKLCH CSS variables in `globals.css`, `tw-animate-css`, and reading the generated source of Button/Dialog/Tabs/Table/etc. to understand composition.
- **Underlying primitives:** Radix UI (https://www.radix-ui.com/docs/overview) — accessibility engines for dialog/dropdown/tabs/popover; class-variance-authority (CVA) for variant APIs; clsx + tailwind-merge for class composition. All three are taught, not black-boxed.

### Motion (motion.dev)
- **Docs:** https://motion.dev/docs/animation
- **Key APIs taught:** `motion` components, `animate`/`whileHover`/`whileInView`, layout animations, `AnimatePresence` (exit transitions), springs vs keyframes, gesture handlers, `useReducedMotion`, page/modal/list transitions, skeletons & optimistic UI. Plus **native CSS transitions** and **React View Transitions** (React 19.2 / Next.js 16) where they're the better tool.

### next/image / next/font
- **Docs:** https://nextjs.org/docs/app/api-reference/components/image · .../components/font
- **Taught in modules 15:** `Image` (src/remotePatterns, sizes/sizes heuristics, placeholders, `fill`, priority/preload behavior in 16.x, unoptimized when), `next/font/google` vs `next/font/local`, `display: swap` strategy, self-hosting, `subset`, text metrics, `FontLoader`.

---

## Testing

### Vitest 5.x
- **Docs:** https://vitest.dev
- **Facts:** 5.0 shipped **September 3, 2026**; requires **Node ≥ 22.12** and Vite ≥ 6.4.
- **Taught:** config (`vitest.config.ts` with `@vitejs/plugin-react`/Next env), `vi.mock`/`vi.spyOn`, test projects for unit vs integration, coverage (`@vitest/coverage-v8`), testing async services, testing Server Functions as plain async functions, testing against a real ephemeral Postgres.

### React Testing Library 16.x
- **Docs:** https://testing-library.com/docs/react-testing-library/intro
- **Taught:** behavior-based queries (`getByRole`, `getByLabelText`, `findBy*`), `user-event`, testing form flows, pending/error states, mocking Server Actions, testing with MSW.

### Playwright 1.x
- **Docs:** https://playwright.dev
- **Taught:** E2E config, test fixtures + auth state (storageState) reuse, traces, testing full user journeys: register → login → create product (upload) → order flow → admin sees it → RBAC negative tests (user cannot access admin routes).

### MSW 2.x
- **Docs:** https://mswjs.io
- **Taught:** mocking *external* APIs (payments, email, image CDN) in unit/integration/E2E, request interception in tests, and the explicit rule: **stop mocking your own server code — test it for real.**

---

## Observability & deployment

### Observability (tool-agnostic, with evaluations)
- Structured logging (JSON), request IDs, error tracking (Sentry-class), metrics (Prometheus/OpenTelemetry concepts), tracing concepts, Web Vitals (RUM), failed Server Action monitoring. Vendor evaluation without vendor lock-in.

### Deployment
- **Next.js deploying docs:** https://nextjs.org/docs/app/getting-started/deploying · **Deploying to Platforms:** https://nextjs.org/docs/app/guides/deploying-to-platforms
- **Docker:** https://docs.docker.com · **Next.js Docker guide:** https://nextjs.org/docs/app/guides/docker
- **Node.js:** https://nodejs.org/docs/latest-v24.x/
- **Taught:** Vercel (serverless Node + edge) vs Docker/Node (long-lived process, your CDN, your secrets) vs other PaaS; what each changes for caching (`cacheHandlers`), filesystem, DB pooling, env vars, migrations, background jobs; CI/CD pipeline (GitHub Actions: lint → typecheck → test → build → migrate → deploy, with preview deploys).

### Internationalization
- **next-intl** (https://next-intl.dev) — evaluated as the current maintained option: localized routing, `getRequestConfig` in the RSC model, formatting (dates/numbers/relative time), RTL, locale-aware metadata.

---

## Verification log (what was checked and where)

| Fact | Source | Checked |
|---|---|---|
| Next.js latest stable = 16.3.6 | https://nextjs.org/docs (header + llms.txt `@doc-version`) | 2026-09-28 |
| 16.x GA date, feature list (Turbopack default, Cache Components, proxy.ts, React 19.2, React Compiler stable, enhanced routing, breaking changes) | https://nextjs.org/blog/next-16 | 2026-09-28 |
| `cacheComponents: true` opt-in; `use cache` data/UI levels; cache key composition; serialization constraints; `'use cache: remote'` / `'use cache: private'` | https://nextjs.org/docs/app/api-reference/directives/use-cache | 2026-09-28 |
| `cacheLife` profiles (default/seconds/minutes/hours/days/weeks/max with stale/revalidate/expire); override config; "call it where the cache is defined" guidance | https://nextjs.org/docs/app/api-reference/functions/cacheLife | 2026-09-28 |
| `cacheTag`, `revalidateTag(tag, profile)` SWR semantics, `updateTag` (Server Actions only, immediate expiry), `revalidatePath`, tag-over-path preference | https://nextjs.org/docs/app/getting-started/revalidating | 2026-09-28 |
| Proxy: `proxy.ts` at root/src, named/default `proxy` export, `matcher`, Node.js runtime, "not for slow data fetching", middleware deprecated for edge | https://nextjs.org/docs/app/getting-started/proxy + next-16 blog | 2026-09-28 |
| Server Functions: `'use server'` (file or function level), async required, POST-only, direct-POST exposure → always auth inside, progressive enhancement, sequential dispatch, actions-as-props | https://nextjs.org/docs/app/getting-started/mutating-data | 2026-09-28 |
| create-next-app prompts (TS, ESLint/Biome/None, React Compiler, Tailwind, src, App Router, alias, AGENTS.md); Node ≥ 20.9; TypeScript ≥ 5.1; Turbopack default; `eslint` script (no `next lint`) | https://nextjs.org/docs/app/getting-started/installation | 2026-09-28 |
| Caching (Previous Model) — `force-cache`, `unstable_cache`, segment config `dynamic`/`fetchCache` | https://nextjs.org/docs/app/guides/caching-without-cache-components | 2026-09-28 |
| Docs index for the documentation map | https://nextjs.org/docs/llms.txt | 2026-09-28 |
| React Compiler 1.0 stable (Oct 2025); Next 16 stable support | https://react.dev + Next 16 blog + ecosystem coverage | 2026-09-28 |
| Node.js 24 active LTS (24.21.x, Sept 2026); 26 Current (LTS Oct 2026) | https://nodejs.org + version trackers | 2026-09-28 |
| Better Auth 1.6.x stable (1.6.26, Aug 2026; weekly releases); Vercel acquisition (Jul 2026); Auth.js maintenance transfer (Sep 2025), v5 never GA'd, 2026 fail-open CVE | better-auth.com blog/releases + ecosystem reporting | 2026-09-28 |
| Drizzle v1 stable (mid-2025); Prisma 7 (Nov 2025) as the competing line | orm.drizzle.team + prisma.io release channels | 2026-09-28 |
| Zod 4 stable (release notes) | https://zod.dev/v4 | 2026-09-28 |
| Tailwind v4 + shadcn/ui v4 compatibility (OKLCH vars, tw-animate-css, CLI generates v4) | tailwindcss.com + ui.shadcn.com | 2026-09-28 |
| Vitest 5.0 (Sep 3, 2026), Node ≥ 22.12, Vite ≥ 6.4 | vitest release notes / npm | 2026-09-28 |
| PostgreSQL 18 (released Sep 25, 2025) | https://www.postgresql.org | 2026-09-28 |
| TanStack Query 5.x current line (5.10x core) | TanStack release channels | 2026-09-28 |

**Rule for the rest of the course:** any API shown in code was cross-checked against the official reference at the time of writing. If Next.js ships a 16.x minor that changes something shown here, the "What changed in modern Next.js" module (`25-mastery/04`) is the living changelog.
