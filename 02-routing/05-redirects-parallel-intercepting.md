# Module 09 — Redirects, Parallel Routes & Intercepting Routes

**Phase 2: Routing · Module 9 of 101**

> **Where do these run?** Config redirects and `redirect()`/`permanentRedirect()` execute `[SERVER]`. Parallel and intercepting routes are *rendering* features of the tree — the slot's content renders server-side and streams like everything else.

---

## 1. Concept — Three distinct "go somewhere else" mechanisms

Teams conflate three different tools. Keep them separate:

| Tool | Layer | Use for |
|---|---|---|
| **Config redirects** (`redirects()` in `next.config.ts`) | Build config | Permanent URL policy: old→new URLs, `/docs/*`→help center, trailing-slash normalization. No logic, fast, applies to all matching requests |
| **`redirect()` / `permanentRedirect()`** (functions) | Server code (pages, layouts, actions, route handlers) | *Conditional* redirects that need request data: auth gates, role gates, "you already have an org → `/dashboard`" |
| **Proxy** (`proxy.ts`) | Network boundary | Request-shaping redirects *before* routing: locale, A/B, deprecated path families, optimistic "no session cookie → `/login`" |

Decision: **static policy → config; needs session/DB → `redirect()`; needs to run before everything else on many paths → proxy.** (The proxy is UX, not security — the layout's `redirect()` is the enforcement.)

## 2. Architecture — parallel & intercepting routes

### Parallel routes: multiple independent slots per URL

A page can render **several slots simultaneously**, each with its own content and its own fallback (`default.js`). Classic use: a dashboard where the main content and a live notifications feed load independently — not as nested Suspense, but as *named* routes.

```
dashboard/
├── layout.tsx            ← receives the slot as a prop (below)
├── page.tsx              ← the main content (rendered as children of the layout)
└── @notifications/
    ├── page.tsx          ← the notifications feed (streamed independently)
    └── default.js        ← skeleton shown while @notifications streams
```

The mechanism is prop-based: the **`page.tsx`** inside a slot (here `@main`) is rendered as the slot's content, and every `@slot` folder's rendered content is passed to the *layout* (and page) **as a prop named after the slot** — so `@notifications` arrives as the `notifications` prop. Each slot's `default.js` is its fallback. In practice you write:

`FILE: src/app/dashboard/layout.tsx` (simplified example — [SERVER])

```tsx
import { Suspense } from 'react'
import { renderParallelSlot } from '@/lib/slots' // thin helper wrapping the framework's slot rendering

export default function DashboardLayout({
  children,
  notifications,
}: {
  children: React.ReactNode             // the page (main content)
  notifications: React.ReactNode        // @notifications slot content (or its default.js)
}) {
  return (
    <div className="grid gap-6 lg:grid-cols-[1fr_320px]">
      <section>{children}</section>
      <aside aria-label="Notifications">
        <Suspense fallback={<NotificationsSkeleton />}>{notifications}</Suspense>
      </aside>
    </div>
  )
}
```

**Rule of thumb:** use parallel routes when two *independent content streams* share a layout and one is meaningfully slower (live feed, async cart). For "page + slow section," a plain `<Suspense>` in the page (module 06) is simpler — parallel routes earn their complexity when the *slots have their own routes* (e.g., `/dashboard` and `/dashboard?feed=archive` swapping slot content).

### Intercepting routes: modal-as-a-route

An intercepting route **intercepts** a sibling route and renders it in a *different* layout — typically a **modal** — while keeping the URL of the underlying route. Files use the `.` prefix: `.modal.tsx`, `.drawer.tsx`, `(.)`.

```
products/
├── [slug]/
│   ├── page.tsx              ← the normal page route
│   └── .modal.tsx            ← intercepts the parent's navigation
```

Behavior: navigating to `/products/abc` (while inside `/products`) renders `.modal.tsx` (the product in a dialog) *over* the current page; the URL is `/products/abc`. Navigating with a clean context (fresh load, or the parent not in the tree) renders the normal `page.tsx`. This gives you **deep-linkable modals**: share `/products/abc` — a returning user lands in the modal, a fresh user lands on the page.

Where the capstone uses it: quick-add from a table row (a full product-edit modal over the list), and the "view full order" overlay on the dashboard (the order detail as a modal; a new browser tab gets the full page).

## 3. Production Code — the redirect set

`FILE: next.config.ts` (production pattern — config redirects)

```ts
const nextConfig: NextConfig = {
  reactCompiler: true,
  cacheComponents: true,
  async redirects() {
    return [
      // Permanent URL policy — no logic, applies before the app boots.
      { source: '/home', destination: '/', permanent: true },
      { source: '/app', destination: '/dashboard', permanent: false },  // soft during migration
      // Legacy docs path family → external help center
      { source: '/docs/:path*', destination: 'https://help.example.com/:path*', permanent: true },
    ]
  },
}
```

`FILE: src/app/(app)/page.tsx` (simplified example — [SERVER], conditional)

```tsx
import { redirect } from 'next/navigation'
import { headers } from 'next/headers'
import { auth } from '@/lib/auth'

// '/' inside (app) — the authenticated root. Redirect to the first sensible place.
export default async function AppRoot() {
  const session = await auth.api.getSession({ headers: await headers() })
  if (!session) redirect('/login')
  redirect('/dashboard')
}
```

`FILE: src/proxy.ts` (production pattern — [SERVER, network boundary])

```ts
import { NextResponse, type NextRequest } from 'next/server'

// proxy.ts runs on the Node.js runtime (Next 16) before routing.
// OPTIMISTIC ONLY: a missing cookie is a UX hint, never an authorization decision.
export function proxy(request: NextRequest) {
  const { pathname } = request.nextUrl
  const isApp = pathname.startsWith('/dashboard') || pathname.startsWith('/settings') || pathname.startsWith('/orders')
  const hasSession = request.cookies.has('better-auth.session_token')
  if (isApp && !hasSession) {
    return NextResponse.redirect(new URL('/login', request.url))
  }
  return NextResponse.next()
}

// Run it only where it can help — keep the proxy hot-path tiny.
export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*', '/orders/:path*', '/admin/:path*'],
}
```

**Open-redirect note (module 19-02):** anywhere you redirect to a *variable* target (`?from=`), validate it: relative paths only, or an allowlisted domain. `new URL(userInput)` without validation is the bug.

## 4. Common Mistakes

| Mistake | Fix |
|---|---|
| "Auth middleware" doing DB lookups in the proxy | The proxy is for header/cookie-level checks; session resolution belongs in the layout (module 10) |
| `redirect()` in a client component | It's a server function; client navigation is `router.push`/`Link` |
| Config redirect with `:path*` that's too greedy | Test the matcher against your real URL list; config redirects are global |
| Parallel routes for "page + one slow widget" | Suspense in the page; parallel routes for *route-level* slot independence |
| Intercepting modal without the non-modal page | Share the same URL → provide both `.modal.tsx` and `page.tsx` (intercepting falls through to the page when nothing is intercepted) |
| `permanent: true` on a redirect you might undo | `permanent` tells clients/crawlers to *remember*; use it only for settled policy |

## 5. Security Notes

- Redirect targets from user input are **open-redirect** vectors (phishing via your domain): relative-only or allowlist; log rejected targets.
- The proxy runs on every matched request: keep it **fast and dumb** — no DB, no heavy auth (official guidance: optimistic checks only).
- `permanentRedirect` on an auth-required path can cache the redirect *for logged-out users* incorrectly; prefer non-permanent for auth-gated redirects.

## 6. Performance Notes

- Config redirects are the cheapest redirect (no app boot) — use them for pure URL policy.
- The proxy's `matcher` is a performance control: every unmatched path skips the proxy entirely.
- Intercepting routes stream the modal like any Suspense content — the page behind is already rendered, so the modal is the only incremental payload.

## 7. Exercise

**Beginner.** Add the three config redirects from §3. Verify each with curl (`-I` flag): 301 vs 307 semantics.

**Intermediate.** Implement `/login` with a `?from=` flow: unauthenticated visit to `/settings` → `/login?from=/settings` → after login, back to `/settings`. Implement the validation for `from` (relative-only). Test the open-redirect attempts (`?from=https://evil.com`, `?from=//evil.com`) and confirm they're rejected.

**Production.** Build the quick-add intercepting route: `/products/manage` (list) + `/products/manage/[id]` with `.modal.tsx` (edit modal) + `page.tsx` (full edit page). Verify: modal over list, fresh-tab deep link → full page, back button closes the modal.

## 8. Architecture Challenge

**Prompt:** You're adding i18n (English + French) to the capstone (module 22-05 walks it). Three proposals: (a) `/en/...` `/fr/...` prefix in routing; (b) proxy-based locale detection rewriting to prefixed paths; (c) no prefixes, `Accept-Language` → header + CDN edge cache variants.

For each: what breaks (caching? SEO? URLs for users?), and what is the *minimum* correct system at 2 locales vs 10?

<details>
<summary>Model answer</summary>
(a) explicit prefixes are the SEO-safe, cacheable default: one URL per locale, hreflang metadata, unambiguous. Cost: every internal link must be locale-aware (a `Link` wrapper / `usePathname`-aware helper). (b) is the *companion* to (a): proxy detects `Accept-Language`/cookie and redirects *only* on the locale-less root or known marketing paths — never rewrites app paths (app paths are session-scoped; rewriting them per request fights the client router and breaks the RSC cache). (c) fails SEO (no separate indexable URLs per locale) and CDN caching (edge would serve language variants of the same URL → cross-locale cache pollution) — acceptable only for non-indexed tools. Minimum at 2 locales: (a) + (b)-on-root + locale-aware Link + `next-intl` routing. At 10 locales you add: per-locale sitemap + hreflang generation (module 14-02), RTL support (module 22-05), and a locale fallback hierarchy — still (a)+(b).
</details>

## 9. Official Documentation

- Redirects (config): https://nextjs.org/docs/app/api-reference/config/next-config-js/redirects
- `redirect()`: https://nextjs.org/docs/app/api-reference/functions/redirect
- `permanentRedirect()`: https://nextjs.org/docs/app/api-reference/functions/permanent-redirect
- Proxy: https://nextjs.org/docs/app/getting-started/proxy
- Parallel routes: https://nextjs.org/docs/app/building-your-application/routing/parallel-routes
- Intercepting routes: https://nextjs.org/docs/app/building-your-application/routing/intercepting-routes

## 10. What You Should Know Before Continuing

- [ ] I can place any redirect in the right layer (config / function / proxy) with a one-line reason
- [ ] I understand the proxy's optimistic role and its matcher
- [ ] I know parallel routes = independent *slots*; intercepting routes = same URL, different layout
- [ ] I can validate a `?from=` target against open redirects
- [ ] I've built a deep-linkable modal with intercepting routes

**Next:** Module 10 — URL State: when state belongs in the URL, React state, or the server.
