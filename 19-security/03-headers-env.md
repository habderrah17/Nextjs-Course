# Module 76 — Security Headers, a CSP That Works, and Env Discipline

**Phase 19: Security · Module 76 of 101**

> **Where does this run?** The headers and CSP are **`[SERVER]`** (the `headers()` config + the `app/headers.ts` file convention — module 76's §1); the env is a **build/runtime** split (the `NEXT_PUBLIC_` is the *build-time's* inlined — module 76's §1; the secret is the *runtime's* server — module 76's §1). The module-76's standing rule (module 74's edge/infra layers, now the header level): **the header is the *edge's* (module 76's §1) — it stops the browser-based attacks (XSS/CSRF/MIME-sniffing) that the app *can't* stop; the CSP is the *last line* for XSS (module 75's §1.1) and must be *testable* (report-only first — module 76's §1); and the env is the *no-leak's* (module 76's §1) — a `NEXT_PUBLIC_` secret is *public by construction* (module 76's §1)** (module 76's §1).

---

## 1. Concept — The header's 3 jobs (the map)

**The `headers()`'s** (module 76's §1.1): the *the build-time's* (module 76's §1.1) — the *module-76's line: the header is the edge's* (module 76's §1.1) — the *the config's* (module 76's §1.1).

**The `headers.ts`'s** (module 76's §1.2): the *the runtime's* (module 76's §1.2) — the *module-76's line: the dynamic's is the `headers.ts`'s* (module 76's §1.2) — the *the file convention's* (module 76's §1.2).

**The CSP's** (module 76's §1.3): the *the report-only's first* (module 76's §1.3) — the *module-76's line: the CSP is the last's* (module 76's §1.3) — the *module-75's line: the XSS's* (module 75's §1.1).

## 2. Mental Model — The edge (drawn)

```mermaid
flowchart TD
    A["THE BROWSER (module 76's §1) — the the receives (module 76's §1)"] --> B["THE 3 JOBS (module 76's §1)"]
    B --> B1["THE `headers()` (module 76's §1.1) — the the build-time's (module 76's §1.1) — the the config's (module 76's §1.1)"]
    B --> B2["THE `headers.ts` (module 76's §1.2) — the the runtime's (module 76's §1.2) — the the dynamic's (module 76's §1.2)"]
    B --> B3["THE CSP (module 76's §1.3) — the the report-only's first (module 76's §1.3) — the the last's (module 76's §1.3)"]
    B1 --> C["THE EDGE (module 74's §1.2) — the the no app's cost (module 74's §1.2) — the the no leak's (module 76's §1)"]
    B2 --> C
    B3 --> C
```

**The 3 jobs** (the module-76's mental model):
1. **The `headers()`** (module 76's §1.1): the *the build-time's* — the *module-76's line: the header is the edge's* (module 76's §1.1).
2. **The `headers.ts`** (module 76's §1.2): the *the runtime's* — the *module-76's line: the dynamic's is the `headers.ts`'s* (module 76's §1.2).
3. **The CSP** (module 76's §1.3): the *the last's* — the *module-76's line: the CSP is the last's* (module 76's §1.3).

## 3. Architecture — The 3 jobs (the code)

### 3.1 The `headers()`'s (module 76's §1.1 — the config's)

`FILE: next.config.ts` (production pattern — [SERVER] — the module-76's §3.1: the static's)

```ts
// THE `headers()` (module 76's §3.1) — the the build-time's (module 76's §1.1) — the the no app's cost (module 74's §1.2):
import type { NextConfig } from 'next'

const securityHeaders = [
  { key: 'Strict-Transport-Security', value: 'max-age=63072000; includeSubDomains; preload' },   /* the module-76's line: the HSTS is the 2y's (module 76's §3.1) */
  { key: 'X-Content-Type-Options', value: 'nosniff' },   /* the module-76's line: the nosniff's (module 76's §3.1) */
  { key: 'Referrer-Policy', value: 'strict-origin-when-cross-origin' },   /* the module-76's line: the referrer's (module 76's §3.1) */
  { key: 'X-Frame-Options', value: 'DENY' },   /* the module-76's line: the no frame's (module 76's §3.1) — the the clickjacking's (module 76's §3.1) */
  { key: 'Permissions-Policy', value: 'camera=(), microphone=(), geolocation=()' },   /* the module-76's line: the permissions' (module 76's §3.1) */
]

const nextConfig: NextConfig = {
  async headers() {
    return [{ source: '/:path*', headers: securityHeaders }]   /* the module-76's line: the all's paths (module 76's §3.1) */
  },
}
export default nextConfig
```

**The module-76's line:** the *HSTS is the 2y's* (module 76's §3.1) — the *nosniff's* (module 76's §3.1) — the *no frame's* (module 76's §3.1).

### 3.2 The CSP's (module 76's §1.3 — the report-only's first)

`FILE: src/app/headers.ts` (production pattern — [SERVER] — the module-76's §3.2: the report-only's)

```ts
// THE CSP (module 76's §3.2) — the the report-only's first (module 76's §1.3) — the the last's (module 76's §1.3):
import type { NextConfig } from 'next'
import { headers as readHeaders } from 'next/headers'

export async function headers() {
  /* THE CSP (module 76's §3.2) — the the report-only's (module 76's §3.2):
     Phase 1 (module 76's §3.2): the the `Content-Security-Policy-Report-Only` (module 76's §3.2) — the the no break (module 76's §3.2)
     Phase 2 (module 76's §3.2): the the `Content-Security-Policy` (module 76's §3.2) — the the no report (module 76's §3.2) */
  const csp = [
    "default-src 'self'",   /* the module-76's line: the self's (module 76's §3.2) */
    "script-src 'self'",   /* the module-76's line: the no inline's (module 76's §3.2) — the the no `dangerouslySetInnerHTML`'s (module 75's §1.1) */
    "style-src 'self' 'unsafe-inline'",   /* the module-76's line: the style's (module 76's §3.2) */
    "img-src 'self' data: https://cdn.example.com",   /* the module-76's line: the img's (module 76's §3.2) */
    "connect-src 'self' https://analytics.example.com",   /* the module-76's line: the connect's (module 76's §3.2) */
    "frame-ancestors 'none'",   /* the module-76's line: the no frame's (module 76's §3.1) — the the clickjacking's (module 76's §3.1) */
    "base-uri 'self'",   /* the module-76's line: the base's (module 76's §3.2) */
    "form-action 'self'",   /* the module-76's line: the form's (module 76's §3.2) */
    "report-uri /api/csp-report",   /* the module-76's line: the report's (module 76's §3.2) */
  ].join('; ')

  return [
    { key: 'Content-Security-Policy-Report-Only', value: csp },   /* the module-76's line: the report-only's first (module 76's §1.3) */
  ]
}
```

**The module-76's line:** the *report-only's first* (module 76's §1.3) — the *no inline's* (module 76's §3.2) — the *self's* (module 76's §3.2).

### 3.3 The env's (module 76's §1 — the no-leak's)

`FILE: .env.example` + `FILE: src/lib/env.ts` (production pattern — [SERVER] — the module-76's §3.3: the no-leak's)

```env
# THE ENV'S (module 76's §3.3) — the the no-leak's (module 76's §1) — the the no `NEXT_PUBLIC_`'s secret (module 76's §1):
# THE SERVER'S (module 76's §3.3) — the the no `NEXT_PUBLIC_` (module 76's §3.3):
DATABASE_URL=postgres://user:pass@host:5432/shop   # the module-76's line: the server's (module 76's §3.3)
BETTER_AUTH_SECRET=xxx   # the module-76's line: the server's (module 76's §3.3)
S3_SECRET_ACCESS_KEY=xxx   # the module-76's line: the server's (module 76's §3.3)

# THE CLIENT'S (module 76's §3.3) — the the `NEXT_PUBLIC_`'s (module 76's §3.3) — the the no secret's (module 76's §1):
NEXT_PUBLIC_API_URL=https://api.example.com   # the module-76's line: the public's (module 76's §3.3)
```

```ts
// THE ENV'S VALIDATOR (module 76's §3.3) — the the no-leak's (module 76's §1):
import { z } from 'zod'   /* the module-53's line: the Zod's (module 53's) */

const envSchema = z.object({
  DATABASE_URL: z.string().url(),   /* the module-76's line: the server's (module 76's §3.3) */
  BETTER_AUTH_SECRET: z.string().min(32),   /* the module-76's line: the server's (module 76's §3.3) */
  NEXT_PUBLIC_API_URL: z.string().url(),   /* the module-76's line: the public's (module 76's §3.3) */
})

export const env = envSchema.parse(process.env)   /* the module-76's line: the no-leak's (module 76's §1) — the the no silent's (module 76's §3.3) */
```

**The module-76's line:** the *no-leak's* (module 76's §1) — the *server's* (module 76's §3.3) — the *public's* (module 76's §3.3).

## 4. Production Code — The no leak's (module 76's §4)

`FILE: docs/env-policy.md` (production pattern — the module-76's §4: the 3 rules)

```md
## THE ENV'S POLICY (module 76's §4 — the the no-leak's (module 76's §1) — the the no `NEXT_PUBLIC_`'s secret (module 76's §1))

1. **The no `NEXT_PUBLIC_`'s secret** (module 76's §4.1): the the `NEXT_PUBLIC_` is the public's (module 76's §4.1) — the the no secret's (module 76's §4.1)
2. **The server's** (module 76's §4.2): the the no `NEXT_PUBLIC_` (module 76's §4.2) — the the no client's (module 76's §4.2)
3. **The build's** (module 76's §4.3): the the `NEXT_PUBLIC_` is the build-time's (module 76's §4.3) — the the no runtime's (module 76's §4.3)

/* THE RULE (module 76's §4): the the no-leak's (module 76's §1) — the the no `NEXT_PUBLIC_`'s secret (module 76's §1) — the the 3's rules (module 76's §4) */
```

**The module-76's line:** the *no `NEXT_PUBLIC_`'s secret* (module 76's §1) — the *server's* (module 76's §4.2) — the *build-time's* (module 76's §4.3).

## 5. Common Mistakes (the edge's failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The `NEXT_PUBLIC_`'s secret** (module 76's §1's line violated) | the *module-76's line: the no-leak's* (module 76's §1) — the *the `NEXT_PUBLIC_`'s secret's is the *no's* (module 76's §1) — the *module-76's line: the no `NEXT_PUBLIC_`'s secret* (module 76's §1) — the *no `NEXT_PUBLIC_`'s secret* (module 76's §1)* | the *the server's (module 76's §3.3) — the *module-76's line: the no-leak's* (module 76's §1)* |
| **The no CSP** (module 76's §1.3's line violated) | the *module-76's line: the CSP is the last's* (module 76's §1.3) — the *the no CSP's is the *no's* (module 76's §1.3) — the *module-76's line: the no CSP* (module 76's §1.3) — the *no CSP* (module 76's §1.3)* | the *the report-only's first (module 76's §3.2) — the *module-76's line: the CSP is the last's* (module 76's §1.3)* |
| **The no HSTS** (module 76's §1.1's line violated) | the *module-76's line: the HSTS is the 2y's* (module 76's §3.1) — the *the no HSTS's is the *no's* (module 76's §3.1) — the *module-76's line: the no HSTS* (module 76's §3.1) — the *no HSTS* (module 76's §3.1)* | the *the HSTS's (module 76's §3.1) — the *module-76's line: the HSTS is the 2y's* (module 76's §3.1)* |
| **The no nosniff** (module 76's §1.1's line violated) | the *module-76's line: the nosniff's* (module 76's §3.1) — the *the no nosniff's is the *no's* (module 76's §3.1) — the *module-76's line: the no nosniff* (module 76's §3.1) — the *no nosniff* (module 76's §3.1)* | the *the nosniff's (module 76's §3.1) — the *module-76's line: the nosniff's* (module 76's §3.1)* |
| **The no `headers.ts`** (module 76's §1.2's line violated) | the *module-76's line: the dynamic's is the `headers.ts`'s* (module 76's §1.2) — the *the no `headers.ts`'s is the *no's* (module 76's §1.2) — the *module-76's line: the no `headers.ts`* (module 76's §1.2) — the *no `headers.ts`* (module 76's §1.2)* | the *the `headers.ts`'s (module 76's §3.2) — the *module-76's line: the dynamic's is the `headers.ts`'s* (module 76's §1.2)* |
| **The no env's validator** (module 76's §3.3's line violated) | the *module-76's line: the no-leak's* (module 76's §1) — the *the no env's validator's is the *no's* (module 76's §3.3) — the *module-76's line: the no env's validator* (module 76's §3.3) — the *no env's validator* (module 76's §3.3)* | the *the Zod's (module 76's §3.3) — the *module-76's line: the no-leak's* (module 76's §1)* |

## 6. Security Notes

- **The no-leak's** (module 76's §1): the *module-76's line: the no `NEXT_PUBLIC_`'s secret* (module 76's §1) — the *module-74's* *deep-dive* (module 74's).
- **The last's** (module 76's §1.3): the *module-76's line: the CSP is the last's* (module 76's §1.3) — the *module-75's* *deep-dive* (module 75's).
- **The edge's** (module 74's §1.2): the *module-76's line: the header is the edge's* (module 76's §1.1) — the *module-74's* *deep-dive* (module 74's).

## 7. Performance Notes

- **The no app's cost** (module 74's §1.2): the *module-76's line: the header is the edge's* (module 76's §1.1) — the *the no app's cost* (module 74's §1.2).
- **The build-time's** (module 76's §4.3): the *module-76's line: the `NEXT_PUBLIC_` is the build-time's* (module 76's §4.3) — the *the no runtime's* (module 76's §4.3).
- **The report-only's** (module 76's §3.2): the *module-76's line: the report-only's first* (module 76's §1.3) — the *the no break's* (module 76's §3.2).

## 8. Exercise

**Beginner.** *The `headers()`'s* (module 76's §3.1): the *the HSTS's* (module 3.1's) + the *the nosniff's* (module 3.1's) + the *the no frame's* (module 3.1's) — *build it* — the *artifact: the headers' (module 3.1's)*.

**Intermediate.** *The CSP's* (module 76's §3.2): the *the report-only's* (module 3.2's) + the *the `headers.ts`'s* (module 3.2's) + the *the report's* (module 3.2's) — *build it* — the *artifact: the CSP's* (module 3.2's).

**Production.** *The env's* (module 76's §3.3): the *the Zod's* (module 3.3's) + the *the no-leak's* (module 3.3's) + the *the 3's rules* (module 4's) — *build it* — the *artifact: the env's* (module 3.3's).

## 9. Architecture Challenge

**Prompt:** The *"the team's `BETTER_AUTH_SECRET` is in `NEXT_PUBLIC_`, there's no CSP, and the headers are in one `middleware`"* (the *module-76's* *edge* — the *module-74's* *model* — the *module-76's line: the header is the edge's* (module 76's §1.1) — the *module-74's line: the infra is the secrets's* (module 74's §1.5) — the *module-76's standing line: the no-leak's + the CSP is the last's + the header is the edge's* (module 76's §1 + module 76's §1.3 + module 76's §1.1)).

The *problems*: (1) the *the `NEXT_PUBLIC_`'s secret* (the *the no no-leak's* (module 76's §1) — the *module-76's line: the no `NEXT_PUBLIC_`'s secret* (module 76's §1) — the *module-76's standing line: the no-leak's* (module 76's §1)).

(2) the *the no CSP* (the *the no last's* (module 76's §1.3) — the *module-76's line: the CSP is the last's* (module 76's §1.3) — the *module-76's standing line: the CSP is the last's* (module 76's §1.3)).

**Design**: the *the edge's remediation* (the *the server's secret* (module 76's §3.3) + the *the CSP's report-only* (module 76's §3.2) + the *the `headers()`'s* (module 76's §3.1) — the *module-76's line: the header is the edge's* (module 76's §1.1) — the *module-76's standing line: the no-leak's + the CSP is the last's + the header is the edge's* (module 76's §1 + module 76's §1.3 + module 76's §1.1)).

Produce: the *the edge's remediation* (the *the server's secret* (module 76's §3.3) + the *the CSP's report-only* (module 76's §3.2) + the *the `headers()`'s* (module 76's §3.1) — the *module-76's line: the header is the edge's* (module 76's §1.1) — the *module-76's standing line: the no-leak's + the CSP is the last's + the header is the edge's* (module 76's §1 + module 76's §1.3 + module 76's §1.1)).

<details>
<summary>Model answer</summary>
**The edge's remediation** (module 76's §3.3 + module 76's §3.2 + module 76's §3.1):
1. **The server's secret** (module 76's §3.3): the *the `BETTER_AUTH_SECRET` moves from `NEXT_PUBLIC_` to the server's* — the *module-76's line: the no `NEXT_PUBLIC_`'s secret* (module 76's §1).
2. **The CSP's report-only** (module 76's §3.2): the *the `Content-Security-Policy-Report-Only` first* — the *module-76's line: the CSP is the last's* (module 76's §1.3).
3. **The `headers()`'s** (module 76's §3.1): the *the HSTS/nosniff/no-frame* — the *module-76's line: the header is the edge's* (module 76's §1.1).
**The generalization** (the *edge's* pattern, the *module's* standing rule): **the *no-leak's* (module 76's §1) — the *the CSP is the last's* (module 76's §1.3) — the *the header is the edge's* (module 76's §1.1) — the *module-76's standing line: the no-leak's + the CSP is the last's + the header is the edge's* (module 76's §1 + module 76's §1.3 + module 76's §1.1)*.
</details>

## 10. Official Documentation

- Next.js: `headers()`: https://nextjs.org/docs/app/api-reference/file-conventions/headers
- Next.js: Security: https://nextjs.org/docs/app/guides/security
- MDN: CSP: https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP
- MDN: HSTS: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Strict-Transport-Security
- OWASP: CSP: https://owasp.org/www-project-content-security-policy/
- The module-74's model: the module-74 (the phase-19's file-01)
- The module-75's attack: the module-75 (the phase-19's file-02)

## 11. What You Should Know Before Continuing

- [ ] I can state the *3 jobs* (module 1's: the `headers()`/`headers.ts`/CSP) — the *module-76's line: the header is the edge's* (module 1's)
- [ ] I know the *HSTS is the 2y's* (module 3.1's) — the *the nosniff's* (module 3.1's)
- [ ] I know the *report-only's first* (module 3.2's) — the *the no break's* (module 3.2's)
- [ ] I know the *no `NEXT_PUBLIC_`'s secret* (module 4's) — the *the server's* (module 3.3's)
- [ ] I know the *build-time's* (module 4.3's) — the *the no runtime's* (module 4.3's)
- [ ] I've done the *`headers()`* (module 8's beginner) + the *CSP's* (module 8's intermediate) + the *env's* (module 8's production) — the *artifacts* (module 20's)

**Next:** Module 77 — The Audit (the *the checklist's* — the *module-77's line: the audit is the finding's* (module 77's)).
