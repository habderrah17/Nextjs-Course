# 03 — Course Roadmap & Progress Tracker

The roadmap in the README is the teaching sequence. This file is the **working tracker**: each module with its files, what it builds into the capstone, and the gate you must pass. Mark them off as you go.

> ⚡ = optional deep-dive (core path never depends on it). All other modules are required.

---

## PHASE 1 — Foundations (mental model → running app)

| # | Module | Files | Gate |
|---|---|---|---|
| 1 | Why Next.js & the execution model | `01-fundamentals/01-why-nextjs.md` | Can explain why an SPA + separate API duplicates work, and where each part of a Next.js app runs |
| 2 | Project setup: `create-next-app` deep dive | `01-fundamentals/02-project-setup.md` | Can create a project and explain **every** CLI option; scaffold exists with `cacheComponents` on |
| 3 | Project structure & why | `01-fundamentals/03-project-structure.md` | Capstone folder tree created; can defend each folder's existence |
| 4 | TypeScript for Next.js | `01-fundamentals/04-typescript-for-nextjs.md` | DTOs, discriminated unions, env typing, typed params in working code |

**Capstone stage after Phase 1:** Stage 1 — scaffold with correct structure, lint + typecheck + test skeleton green.

## PHASE 2 — Routing (the App Router, deeply)

| # | Module | Files | Gate |
|---|---|---|---|
| 5 | App Router deep dive (route segments, special files) | `02-routing/01-app-router-deep-dive.md` | Can draw the route tree → layout tree → HTML |
| 6 | Layouts, nesting, route groups, templates | `02-routing/02-layouts-nesting-route-groups.md` | Marketing vs app layout split with distinct `loading`/`error` |
| 7 | Dynamic & catch-all routes | `02-routing/03-dynamic-catch-all-routes.md` | Product `[slug]`, blog `[...slug]`, `generateStaticParams` + fallbacks |
| 8 | `loading.tsx` / `error.tsx` / `not-found.tsx` | `02-routing/04-special-files.md` | Every route has defined loading/error/not-found behavior |
| 9 | Redirects, parallel routes, intercepting routes | `02-routing/05-redirects-parallel-intercepting.md` | `(marketing)/(app)` groups; modal-as-route demo |
| 10 | URL state (search, filters, pagination, tabs) | `02-routing/06-url-state.md` | Can decide URL vs React state vs server state for 10 given cases |
| 11 | Instant Navigation & prefetching | `02-routing/07-instant-navigation.md` | Can explain link hover prefetch, layout dedup, incremental prefetch, and the diff vs SPA navigation |

**Capstone stage after Phase 2:** Stage 2 — public site skeleton: landing, pricing, blog index/detail, catalog routes, all with loading/error/not-found.

## PHASE 3 — Server/Client Components (the most important module cluster)

| # | Module | Files | Gate |
|---|---|---|---|
| 12 | Server Components deep dive | `03-server-client-components/01-server-components-deep-dive.md` | Can explain the RSC wire format at a high level and what never ships |
| 13 | Client Components & the boundary | `03-server-client-components/02-client-components-boundaries.md` | Can mark every file in the current app [SERVER]/[CLIENT] |
| 14 | Serialization & what can cross | `03-server-client-components/03-serialization.md` | Lists the serializable set; catches an unserializable prop |
| 15 | Server-first architecture (bad vs good) | `03-server-client-components/04-server-first-architecture.md` | Refactors a "use client page" into server page + client islands |

**Capstone stage after Phase 3:** Stage 3 — design system + public pages rebuilt server-first with a11y. (ARCHITECTURE REVIEW #1)

## PHASE 4 — Data Fetching

| # | Module | Files | Gate |
|---|---|---|---|
| 16 | Fetching in Server Components | `04-data-fetching/01-fetching-in-server-components.md` | Direct DB reads in pages; `fetch` for external APIs |
| 17 | The service layer & direct DB access | `04-data-fetching/02-service-layer.md` | Zero ORM code outside `services/`; DTOs everywhere |
| 18 | Parallel fetching & waterfalls | `04-data-fetching/03-parallel-and-waterfalls.md` | `Promise.all`/`allSettled` patterns; a waterfall found & killed |
| 19 | The request lifecycle | `04-data-fetching/04-request-lifecycle.md` | Can trace a dashboard request through all four execution times |

**Capstone stage after Phase 4:** Stage 4 — Postgres + Drizzle schema; catalog + dashboard data live.

## PHASE 5 — Caching (deep)

| # | Module | Files | Gate |
|---|---|---|---|
| 20 | The caching mental model (6 layers) | `05-caching/01-caching-mental-model.md` | Answers the 5 caching questions for 5 real data pieces |
| 21 | Cache Components deep dive | `05-caching/02-cache-components.md` | Data-level + UI-level `'use cache'`; cache keys understood |
| 22 | `cacheLife` profiles | `05-caching/03-cachelife.md` | Assigns a profile to every cached surface in the app, with reasoning |
| 23 | Revalidation: `updateTag` / `revalidateTag` / `revalidatePath` | `05-caching/04-revalidation.md` | Implements a mutation with the *correct* invalidation primitive |
| 24 | Static vs dynamic rendering (why, not "static is faster") | `05-caching/05-static-vs-dynamic.md` | Explains build-time vs request-time vs the equation with caching |
| 25 | **Dashboard cache case study** | `05-caching/06-dashboard-case-study.md` | The full per-section caching plan for the capstone dashboard |
| 26 ⚡ | The previous model (fetch cache / `unstable_cache`) & migration | `05-caching/07-previous-model.md` | Can maintain a Next 14/15 codebase and migrate it |

**Capstone stage after Phase 5:** Stage 10 — full caching strategy applied; invalidation verified in dev. (ARCHITECTURE REVIEW #2)

## PHASE 6 — Streaming & Suspense

| # | Module | Files | Gate |
|---|---|---|---|
| 27 | Streaming deep dive | `06-streaming-suspense/01-streaming-deep-dive.md` | Explains shell + dynamic holes + nested boundaries |
| 28 | **Dashboard streaming lab** | `06-streaming-suspense/02-dashboard-lab.md` | Revenue/Orders/Activity/Analytics load independently with skeletons |

**Capstone stage after Phase 6:** Dashboard streams all sections independently.

## PHASE 7 — Server Functions / Actions (mutations)

| # | Module | Files | Gate |
|---|---|---|---|
| 29 | Server Functions vs Server Actions (official terminology) | `07-server-functions/01-server-functions-actions.md` | Creates, invokes (form + event handler), explains POST-only & sequential dispatch |
| 30 | Forms & progressive enhancement | `07-server-functions/02-forms-and-enhancement.md` | A form that works with JS disabled |
| 31 | Validation & error round-trips | `07-server-functions/03-validation-errors.md` | Zod in actions; field + form errors returned to the form |
| 32 | Optimistic UI & pending states | `07-server-functions/04-optimistic-pending.md` | `useOptimistic` + `useFormStatus` + rollback on server error |
| 33 | Decision matrix: Action vs Route Handler vs external API | `07-server-functions/05-decision-matrix.md` | Chooses correctly for 10 scenarios |

**Capstone stage after Phase 7:** Stage 5–8 mutation paths (products, orders, profile, settings).

## PHASE 8 — Route Handlers & BFF

| # | Module | Files | Gate |
|---|---|---|---|
| 34 | Route Handlers deep dive | `08-route-handlers-bff/01-route-handlers.md` | GET/POST/PUT/PATCH/DELETE, headers/cookies, status codes, streaming responses |
| 35 | BFF architecture | `08-route-handlers-bff/02-bff.md` | Draws the BFF topology; knows when to add the layer |
| 36 | API design & response contracts | `08-route-handlers-bff/03-api-design.md` | Consistent response envelope, error format, pagination, auth for a public API |

**Capstone stage after Phase 8:** Public API (products read) + webhook receiver + CSV export stream.

## PHASE 9 — Database & ORM

| # | Module | Files | Gate |
|---|---|---|---|
| 37 | Postgres + Drizzle setup | `09-database/01-setup.md` | Local Postgres (Docker), Drizzle configured, `server-only` guard, env |
| 38 | Schema design & migrations | `09-database/02-schema-migrations.md` | Capstone data model with constraints, indexes, enums |
| 39 | Queries & relations | `09-database/03-queries-relations.md` | Joins, filters, typed relations, N+1 spotted |
| 40 | Transactions, pagination, indexes | `09-database/04-transactions-pagination-indexes.md` | Multi-statement transactions; offset + cursor pagination |
| 41 | Connections & serverless pooling | `09-database/05-connections-serverless.md` | Explains why serverless breaks naive pools; configures pooler |
| 42 ⚡ | Prisma alternative | `09-database/06-prisma.md` | Can maintain/build the same app on Prisma |

**Capstone stage after Phase 9:** Stage 4 complete (data model) + Stage 9 groundwork.

## PHASE 10 — Authentication

| # | Module | Files | Gate |
|---|---|---|---|
| 43 | Identity / authentication concepts (library-independent) | `10-authentication/01-concepts.md` | Explains session lifecycle end-to-end without a library name |
| 44 | Better Auth architecture & setup | `10-authentication/02-better-auth.md` | Schema in your DB, adapter wired, `auth.ts` server client |
| 45 | Sessions & cookie security | `10-authentication/03-sessions-cookies.md` | Can explain HttpOnly/Secure/SameSite/rotation/revocation and set them correctly |
| 46 | Core flows | `10-authentication/04-flows.md` | Login, register, logout, password reset, email verification, MFA (TOTP concept) |
| 47 | Auth with Cache Components | `10-authentication/05-with-cache-components.md` | Reads session without blocking the shell; caches session-derived data safely |

**Capstone stage after Phase 10:** Stage 5 — real auth on the platform.

## PHASE 11 — Authorization & RBAC

| # | Module | Files | Gate |
|---|---|---|---|
| 48 | RBAC + policy architecture | `11-authorization/01-rbac.md` | Roles → permissions → policy checks in services |
| 49 | **Multi-tenancy** | `11-authorization/02-multi-tenancy.md` | Org/membership/roles; cross-tenant access prevented (and tested) |
| 50 | Frontend vs server boundaries; 401 vs 403 | `11-authorization/03-boundaries.md` | UI affordances vs enforcement; correct status codes everywhere |

**Capstone stage after Phase 11:** Stage 9 — admin console with server-enforced RBAC. (ARCHITECTURE REVIEW #3)

## PHASE 12 — Forms & Validation

| # | Module | Files | Gate |
|---|---|---|---|
| 51 | Forms deep dive (React + actions) | `12-forms-validation/01-forms.md` | field errors, form errors, pending, duplicate-submission, a11y |
| 52 | React Hook Form + Zod | `12-forms-validation/02-rhf-zod.md` | Shared schema client+server; resolver wired |
| 53 | Server validation round-trip | `12-forms-validation/03-server-roundtrip.md` | Client-invalid → server-invalid → reset with field errors |
| 54 | Advanced forms | `12-forms-validation/04-advanced.md` | Checkout, product create/edit, admin user management |

**Capstone stage after Phase 12:** All forms complete.

## PHASE 13 — UI & Design System

| # | Module | Files | Gate |
|---|---|---|---|
| 55 | Tailwind v4 | `13-ui-design-system/01-tailwind-v4.md` | CSS-first config, `@theme`, OKLCH, dark variant |
| 56 | shadcn/ui architecture (read the source) | `13-ui-design-system/02-shadcn.md` | Explains registry model, CVA, Radix, class composition; extends a component |
| 57 | Design tokens & theming | `13-ui-design-system/03-tokens.md` | Full token set: color/spacing/typography/radius/shadow/breakpoint; dark mode |
| 58 | Component patterns & states | `13-ui-design-system/04-components.md` | Button→Toast built: default/hover/focus/disabled/loading/empty/error |
| 59 | Animation (Motion + View Transitions + CSS) | `13-ui-design-system/05-animation.md` | Page/modal/list/hover transitions; reduced-motion respected |
| 60 | Accessibility (throughout) | `13-ui-design-system/06-accessibility.md` | Keyboard, focus, ARIA, contrast, screen-reader pass on every component |

**Capstone stage after Phase 13:** Stage 3 complete — design system is the app's skeleton.

## PHASE 14 — SEO

| # | Module | Files | Gate |
|---|---|---|---|
| 61 | Metadata system | `14-seo/01-metadata.md` | static + `generateMetadata`, canonical, OG/Twitter, OG image generation |
| 62 | Structured data, sitemap, robots | `14-seo/02-structured-sitemap-robots.md` | JSON-LD for products/articles; `sitemap.ts`; `robots.ts` |
| 63 | SEO case studies | `14-seo/03-case-studies.md` | Product page + article page fully optimized; URL strategy |

**Capstone stage after Phase 14:** Stage 12 — SEO complete.

## PHASE 15 — Images & Fonts

| # | Module | Files | Gate |
|---|---|---|---|
| 64 | `next/image` deep dive | `15-images-fonts/01-images.md` | Responsive, remote patterns, priorities, when NOT to use it |
| 65 | `next/font` deep dive | `15-images-fonts/02-fonts.md` | Google vs local, `display`, self-hosting, typography scale |

**Capstone stage after Phase 15:** Product galleries + optimized typography.

## PHASE 16 — File Uploads

| # | Module | Files | Gate |
|---|---|---|---|
| 66 | Uploads deep dive | `16-file-uploads/01-uploads.md` | Multipart in actions/route handlers, size/MIME validation, object storage, signed URLs, security |

**Capstone stage after Phase 16:** Product image uploads end-to-end.

## PHASE 17 — Data-Heavy Screens

| # | Module | Files | Gate |
|---|---|---|---|
| 67 | Search / filter / sort / pagination (URL-driven) | `17-data-screens/01-data-screens.md` | Products, Orders, Users tables: server-side everything, cursor concepts, debounce where needed |
| 68 | Dashboard architecture (SC/CC split) | `17-data-screens/02-dashboard.md` | Sidebar/topbar/breadcrumbs/charts/stats; the server/client division defended |

**Capstone stage after Phase 17:** Stages 7–9 screens complete. (ARCHITECTURE REVIEW #4)

## PHASE 18 — Performance

| # | Module | Files | Gate |
|---|---|---|---|
| 69 | Measure first | `18-performance/01-measure-first.md` | Instrumented: dev logs, Web Vitals, bundle report, `EXPLAIN ANALYZE` |
| 70 | Server performance | `18-performance/02-server.md` | TTFB, waterfalls, cache hits, connection reuse |
| 71 | Client performance | `18-performance/03-client.md` | Bundle budgets, client JS reduction, hydration, React Compiler |
| 72 | Database performance | `18-performance/04-database.md` | N+1, indexes, query plans, pool sizing |
| 73 | Web Vitals in the loop | `18-performance/05-web-vitals.md` | LCP/INP/CLS (+ TTFB) targets at p75 + field instrumentation loop |

**Capstone stage after Phase 18:** Stage 14 — performance pass with before/after numbers.

## PHASE 19 — Security

| # | Module | Files | Gate |
|---|---|---|---|
| 74 | Security model & responsibilities | `19-security/01-model.md` | Maps every threat to a layer (client/server/DB/infra) |
| 75 | The attack surface | `19-security/02-attacks.md` | XSS, CSRF, SSRF, CORS, SQLi, open redirect, IDOR, auth attacks, upload abuse — each with the Next.js-specific vector and fix |
| 76 | Headers, CSP, environment variables | `19-security/03-headers-env.md` | Security headers via proxy/headers; CSP that works; env discipline (build vs runtime, `NEXT_PUBLIC_` leaks) |
| 77 | The audit checklist | `19-security/04-audit.md` | Runs the checklist on the capstone and fixes findings |

**Capstone stage after Phase 19:** Stage 15 — security hardening complete.

## PHASE 20 — Testing

| # | Module | Files | Gate |
|---|---|---|---|
| 78 | Testing strategy & pyramid | `20-testing/01-strategy.md` | Decides what gets which test level and why |
| 79 | Vitest + RTL (unit/integration) | `20-testing/02-vitest-rtl.md` | Services, Server Functions, components, route handlers tested; real Postgres in integration |
| 80 | Playwright E2E | `20-testing/03-playwright.md` | Full journeys incl. auth, authz negatives, CRUD, navigation, upload |
| 81 | MSW & when mocking stops | `20-testing/04-msw.md` | External APIs mocked; own server never mocked |

**Capstone stage after Phase 20:** Stage 13 — test suite green in CI.

## PHASE 21 — Observability & Analytics

| # | Module | Files | Gate |
|---|---|---|---|
| 82 | Observability | `21-observability-analytics/01-observability.md` | Structured logs + request IDs, error tracking, metrics, tracing concepts, failed-action alerts |
| 83 | Analytics | `21-observability-analytics/02-analytics.md` | Page/view events, conversions, privacy, performance cost |

**Capstone stage after Phase 21:** Monitoring live.

## PHASE 22 — Deployment

| # | Module | Files | Gate |
|---|---|---|---|
| 84 | Deployment targets | `22-deployment/01-targets.md` | Vercel vs Docker/Node vs other — with the caching/fs/connections/env implications of each |
| 85 | Node vs Edge runtimes | `22-deployment/02-runtimes.md` | `runtime` export, compatibility matrix, cold starts |
| 86 | Docker & CI/CD | `22-deployment/03-docker-cicd.md` | Multi-stage Dockerfile; GitHub Actions: lint→type→test→build→migrate→deploy; preview deploys |
| 87 ⚡ | Background jobs | `22-deployment/04-background-jobs.md` | Why not in-request; queues/workers conceptually, one real worker implemented |
| 88 ⚡ | Internationalization | `22-deployment/05-i18n.md` | next-intl: localized routes, formatting, RTL, locale metadata |

**Capstone stage after Phase 22:** Stage 15 — deployed with CI/CD. (ARCHITECTURE REVIEW #5)

## PHASE 23 — Production Architecture

| # | Module | Files | Gate |
|---|---|---|---|
| 89 | The four backend architectures | `23-production-architecture/01-architectures.md` | Next-only / BFF+backend / frontend+API / microservices — when each wins |
| 90 | State management, decided | `23-production-architecture/02-state.md` | The 5-step preference ladder applied to the capstone; one Zustand store with justification |
| 91 | TanStack Query, decided | `23-production-architecture/03-tanstack-query.md` | Server cache vs client cache; one justified use in the capstone (or a documented "not needed") |
| 92 | Architecture review checklist | `23-production-architecture/04-reviews.md` | The recurring review — used at all 5 review points |

## PHASE 24 — Capstone

| # | Module | Files | Gate |
|---|---|---|---|
| 93 | **PRD** (requirements only) | `24-capstone/01-prd.md` | A senior PM could hand this to a team |
| 94 | Design workshop (you design, then review) | `24-capstone/02-design-workshop.md` | 9 design questions answered in writing |
| 95 | Staged build (15 stages) | `24-capstone/03-staged-build.md` | Folder tree + data-flow + boundary audit at every stage |
| 96 | Senior architect review | `24-capstone/04-architect-review.md` | The review pass over your design |

## PHASE 25 — Mastery (synthesis)

| # | Module | Files | Gate |
|---|---|---|---|
| 97 | **Debugging lab: 15 broken apps** | `25-mastery/01-debugging-lab.md` | Diagnose → fix → explain mental model for all 15 |
| 98 | Anti-patterns compendium | `25-mastery/02-anti-patterns.md` | The full catalog: bad patterns, common mistakes, outdated patterns, security/perf warnings |
| 99 | Tradeoffs compendium | `25-mastery/03-tradeoffs.md` | Every WHEN/WHY/COST from the course in one reference |
| 100 | "What changed in modern Next.js" changelog | `25-mastery/04-changelog.md` | Old vs modern for every breaking change, verified in official docs |
| 101 | **Official documentation map** | `25-mastery/05-documentation-map.md` | Every technology → official docs → the specific pages this course used |

---

## Progress log (write dates)

| Phase | Done | Date |
|---|---|---|
| 1 Foundations | ✅ | 2026-09-28 |
| 2 Routing | ✅ | 2026-09-28 |
| 3 Server/Client Components | ✅ | 2026-09-28 |
| 4 Data Fetching | ✅ | 2026-09-28 |
| 5 Caching | ✅ | 2026-09-28 |
| 6 Streaming | ✅ | 2026-09-28 |
| 7 Server Functions | ✅ | 2026-09-28 |
| 8 Route Handlers & BFF | ✅ | 2026-09-28 |
| 9 Database | ✅ | 2026-09-28 |
| 10 Authentication | ✅ | 2026-09-28 |
| 11 Authorization | ✅ | 2026-09-28 |
| 12 Forms | ✅ | 2026-09-28 |
| 13 UI & Design System | ✅ | 2026-09-28 |
| 14 SEO | ✅ | 2026-09-28 |
| 15 Images & Fonts | ✅ | 2026-09-28 |
| 16 File Uploads | ✅ | 2026-09-28 |
| 17 Data Screens | ✅ | 2026-09-28 |
| 18 Performance | ✅ | 2026-09-28 |
| 19 Security | ✅ | 2026-09-28 |
| 20 Testing | ✅ | 2026-09-28 |
| 21 Observability & Analytics | ✅ | 2026-09-28 |
| 22 Deployment | ✅ | 2026-09-28 |
| 23 Production Architecture | ✅ | 2026-09-28 |
| 24 Capstone | ✅ | 2026-09-28 |
| 25 Mastery | ✅ | 2026-09-28 |
