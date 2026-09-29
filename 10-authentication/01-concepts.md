# Module 43 — Authentication Concepts: The Session Is a Pointer to a Server-Side Fact

**Phase 10: Authentication · Module 43 of 101**

> **Where does this run?** Everything in this module is **`[SERVER]`** (the session row, the verification, the cookie *setting*) — with a **`[BOTH / BOUNDARY]`** at the cookie itself (the *server* sets it, the *browser* sends it, the *server* reads it). No library name yet — module 44 picks Better Auth. A session is **a pointer to a server-side fact** (module 43's §1) — the cookie is *the handle*, the row is *the truth*; and **never** a token you carry in `localStorage` (module 43's §5, the course's standing rule).

---

## 1. Concept — The session is a pointer (not the fact)

**The three questions** (module 43's §1): *Authentication* = **who are you?** (module 43's §1) — *Authorization* = **what may you do?** (module 50's, Phase 11) — *Session* = **how does the server remember the answer between requests?** (module 43's §1) — the *module-43's line: authN is the who; authZ is the what; the session is the memory* (module 43's §1).

**The session as pointer** (module 43's §1): the *the cookie holds the *handle* (the session id) — the row holds the *truth* (the user_id, the expiry, the device) — the *module-43's line: the cookie is the handle, the row is the truth* (module 43's §1) — the *no self-contained token in the cookie* (module 43's §1) — the *the server can *revoke* (the row's delete (module 43's §1) — the *the token's *no* (module 43's §1)).

**The stateful vs stateless** (module 43's §1.1): the *the stateful (the row) is the *web app's default* (module 43's §1.1) — the *the stateless (the JWT) is the *mobile/edge* (module 43's §1.1) — the *module-43's line: the stateful is the web default* (module 43's §1.1) — the *the stateless is the mobile/edge* (module 43's §1.1) — the *no JWT in the browser's `localStorage`* (module 43's §5).

## 2. Mental Model — The session lifecycle (end-to-end, drawn)

```mermaid
sequenceDiagram
    participant B as the Browser [CLIENT]
    participant S as the Server [SERVER]
    participant D as the DB [SERVER]

    Note over B,D: — LOGIN (the prove) —
    B->>S: POST /login { email, password }
    S->>D: verify the credential (the hash check)
    D-->>S: OK
    S->>D: INSERT session { id, user_id, expires_at, ip, ua }
    D-->>S: the row
    S-->>B: 303 + Set-Cookie: session=<handle>; HttpOnly; Secure; SameSite=Lax

    Note over B,D: — EVERY REQUEST (the remember) —
    B->>S: GET /dashboard + Cookie: session=<handle>
    S->>D: SELECT session WHERE id = <handle> AND expires_at > now()
    D-->>S: the row (the user_id)
    S-->>B: the HTML (the user's)

    Note over B,D: — ROTATION (the refresh) —
    B->>S: GET + the old handle
    S->>D: UPDATE session SET expires_at = now() + 7d WHERE id = <handle> (the age refresh)
    S-->>B: 200

    Note over B,D: — LOGOUT / REVOCATION (the forget) —
    B->>S: POST /logout
    S->>D: DELETE session WHERE id = <handle>
    S-->>B: 303 + Set-Cookie: session= (the expired)
    B->>S: GET + the dead handle
    S-->>B: 401 (the redirect to /login)
```

**The 5 verbs** (the module-43's mental model):
1. **The prove** (module 43's §2.1): the *the credential's check* (the hash) — the *the session's create* (the row's `INSERT`).
2. **The remember** (module 43's §2.2): the *the cookie's send* — the *the row's lookup* — the *the user's resolve* (the *every request* (module 2's execution model: the 4 execution times, the request-time)).
3. **The refresh** (module 43's §2.3): the *the expiry's extend* (the *sliding* window) — the *the `expires_at`'s update* (module 43's §2.3).
4. **The forget** (module 43's §2.4): the *the row's `DELETE`* — the *the cookie's expire* (the *logout*).
5. **The revoke** (module 43's §2.5): the *the *admin's* `DELETE`* (the *no logout* — the *the security incident's* (module 43's §2.5) — the *module-43's line: the revoke is the no-logout logout* (module 43's §2.5)).

## 3. Architecture — The session row (the shape, library-independent)

`FILE: the session table (the shape — [SERVER] — the module-43's §3: the column set every session store needs)` (simplified example)

| Column | The type | The job | The module-43's line |
|---|---|---|---|
| **`id`** | `uuid` | the *handle* (the cookie's value) | the *the `id` is the handle* (module 43's §3) — the *the no `user_id` in the cookie* (module 43's §3) |
| **`user_id`** | `uuid` (FK) | the *truth* (the who) | the *the `user_id` is the truth* (module 43's §3) |
| **`expires_at`** | `timestamptz` | the *when* (the expiry) | the *the `expires_at` is the when* (module 43's §3) — the *the `now()` check is the lookup's* (module 43's §2.2) |
| **`created_at`** | `timestamptz` | the *birth* (the absolute cap) | the *the `created_at` is the birth* (module 43's §3) — the *the no infinite slide* (module 43's §3) |
| **`ip_address`** | `varchar` | the *where* (the anomaly's signal) | the *the `ip_address` is the where* (module 43's §3) — the *module-19-02's attack's* (module 75's) |
| **`user_agent`** | `varchar` | the *what device* (the anomaly's signal) | the *the `user_agent` is the what device* (module 43's §3) |
| **`active`** | `boolean` (optional) | the *the 2FA's pending* (module 46's MFA) | the *the `active` is the 2FA's pending* (module 46's) |

**The module-43's line:** the *the row is the truth* (module 43's §1) — the *the 7 columns are the session's* (module 43's §3) — the *the cookie is the handle* (module 43's §1).

## 4. Production Code — The session read (the shape, the code)

`FILE: src/lib/session.ts` (the shape — [SERVER] — the module-43's §4: the single entry point, the module-17's service rule's sibling)

```ts
// THE SESSION READ (module 43's §4 — the single entry point — the module-17's service rule's sibling (module 17's rule 1)):
import 'server-only'

// THE SHAPE (module 43's §4): the session's read is the ONE function (module 43's §4) — the no inline's cookie's read (module 43's §4):
export type Session = {
  userId: string      // the module-43's §3's truth (module 43's §3)
  expiresAt: Date
  // the module-50's (Phase 11): the role (module 11's RBAC) — the orgId (module 11's tenancy) — added there
} | null

export async function getSession(): Promise<Session> {
  // THE IMPLEMENTATION (module 44's: the Better Auth's auth.api.getSession (module 44's) — the module-43's line: the shape is the module-44's impl (module 44's)):
  // const s = await auth.api.getSession({ headers: await headers() })
  // return s ? { userId: s.user.id, expiresAt: s.expiresAt } : null
  throw new Error('module 44 implements this')   // the module-43's line: the concept is the shape — the module-44's is the impl (module 44's)
}
```

**The module-43's line:** the *session's read is the ONE function* (module 43's §4) — the *the no inline's cookie's read* (module 43's §4) — the *module-44's is the impl* (module 44's).

## 5. Common Mistakes (the auth failures, library-independent)

| Mistake | The symptom | Fix |
|---|---|---|
| **The `localStorage` token** (module 43's §5's line: the no self-contained token in the cookie) | the *module-43's line: the stateful is the web default* (module 43's §1.1) — the *the `localStorage` is the *XSS's* (module 43's §5) — the *module-19-02's XSS* (module 75's) — the *module-43's line: the `localStorage` is the XSS's* (module 43's §5) — the *no `localStorage` token* (module 43's §5)* | the *the `HttpOnly` cookie* (module 45's) — the *module-43's line: the stateful is the web default* (module 43's §1.1)* |
| **The stateless JWT in the browser** (module 43's §1.1's line violated) | the *module-43's line: the stateless is the mobile/edge* (module 43's §1.1) — the *the JWT in the browser is the *no-revoke* (module 43's §1.1) — the *module-43's line: the stateless is the mobile/edge* (module 43's §1.1) — the *no JWT in the browser* (module 43's §1.1)* | the *the stateful's row* (module 43's §1.1) — the *module-43's line: the stateful is the web default* (module 43's §1.1)* |
| **The no `expires_at`** (module 43's §3's line violated) | the *module-43's line: the `expires_at` is the when* (module 43's §3) — the *the no `expires_at` is the *infinite* (module 43's §3) — the *module-43's line: the `expires_at` is the when* (module 43's §3) — the *no infinite session* (module 43's §3)* | the *the `expires_at`* (module 43's §3) + the *the `created_at`'s cap* (module 43's §3) |
| **The no revocation** (module 43's §2.5's line violated: the no `DELETE`) | the *module-43's line: the revoke is the no-logout logout* (module 43's §2.5) — the *the no revocation is the *stolen handle's forever* (module 43's §2.5) — the *module-43's line: the revoke is the no-logout logout* (module 43's §2.5) — the *no stolen handle's forever* (module 43's §2.5)* | the *the `DELETE`* (module 43's §2.5) — the *module-43's line: the revoke is the no-logout logout* (module 43's §2.5)* |
| **The `user_id` in the cookie** (module 43's §3's line violated) | the *module-43's line: the no `user_id` in the cookie* (module 43's §3) — the *the `user_id` in the cookie is the *spoof* (module 43's §3) — the *module-43's line: the no `user_id` in the cookie* (module 43's §3) — the *no spoof* (module 43's §3)* | the *the session id in the cookie* (module 43's §3) — the *the row's lookup* (module 43's §2.2) |
| **The session read inline** (module 43's §4's line violated: the no single entry point) | the *module-43's line: the session's read is the ONE function* (module 43's §4) — the *the inline's cookie's read is the *drift* (module 43's §4) — the *module-43's line: the session's read is the ONE function* (module 43's §4) — the *no inline's cookie's read* (module 43's §4)* | the *the `getSession()`* (module 43's §4) — the *module-43's line: the session's read is the ONE function* (module 43's §4)* |
| **The 401 vs 403 confusion** (module 50's, Phase 11) | the *module-50's line: the 401 is the who* (module 50's) — the *the 403 is the *what* (module 50's) — the *module-43's line: the 401 is the who, the 403 is the what* (module 50's) — the *module-50's* *deep-dive* (module 50's)* | the *module-50's* *the 401/403* (module 50's) — the *module-50's deep-dive* (module 50's) |

## 6. Security Notes

- **The `HttpOnly` is the XSS's guard** (module 45's): the *module-45's line: the `HttpOnly` is the XSS's guard* (module 45's) — the *module-43's line: the `HttpOnly` is the XSS's guard* (module 45's) — the *module-45's* *deep-dive* (module 45's).
- **The `Secure` is the HTTPS's** (module 45's): the *module-45's line: the `Secure` is the HTTPS's* (module 45's) — the *module-43's line: the `Secure` is the HTTPS's* (module 45's).
- **The `SameSite=Lax` is the CSRF's guard** (module 45's): the *module-45's line: the `SameSite=Lax` is the CSRF's guard* (module 45's) — the *module-43's line: the `SameSite=Lax` is the CSRF's guard* (module 45's).
- **The revocation is the incident's** (module 43's §2.5): the *module-43's line: the revoke is the no-logout logout* (module 43's §2.5) — the *the stolen handle's forever is the incident's* (module 43's §2.5).
- **The `ip_address`/`user_agent` is the anomaly's** (module 43's §3): the *module-43's line: the `ip_address` is the where* (module 43's §3) — the *the anomaly's signal* (module 43's §3) — the *module-75's* *attack's* (module 75's).

## 7. Performance Notes

- **The session lookup is the per-request** (module 43's §2.2): the *the row's lookup is the *per-request* (module 43's §2.2) — the *module-43's line: the session lookup is the per-request* (module 43's §2.2) — the *the cache's* (module 20's L3) — the *module-20's* *deep-dive* (module 20's).
- **The `proxy`'s cookie check is the fast** (module 44's): the *module-44's line: the `proxy`'s cookie check is the fast* (module 44's) — the *module-43's line: the `proxy`'s cookie check is the fast* (module 44's) — the *module-44's* *deep-dive* (module 44's).
- **The `use cache: private` is the session's** (module 47's): the *module-47's line: the `use cache: private` is the session's* (module 47's) — the *module-43's line: the `use cache: private` is the session's* (module 47's) — the *module-47's* *deep-dive* (module 47's).

## 8. Exercise

**Beginner.** *The lifecycle* (module 43's §2): the *the sequence diagram* (module 43's §2) — the *the 5 verbs* (module 2's) — *draw it* (the login → the request → the refresh → the logout → the revoke) — the *artifact: the 5-verb's diagram* (module 20's).

**Intermediate.** *The session row* (module 43's §3): the *the 7 columns* (module 3's) — the *the no `user_id` in the cookie* (module 3's) — the *the no infinite session* (module 3's) — *design it* — the *artifact: the session table's DDL* (module 20's).

**Production.** *The revocation* (module 43's §2.5): the *the no revocation is the stolen handle's forever* (module 2.5's) — the *the `DELETE`* (module 2.5's) — the *the admin's no-logout* (module 2.5's) — *implement it* — the *artifact: the revocation's endpoint* (module 20's).

## 9. Architecture Challenge

**Prompt:** The *"the team wants to put the JWT in the `localStorage` for the mobile's API"* (the *module-43's* *stateless* — the *module-43's §5's line: the no self-contained token in the cookie* — the *module-19-02's XSS* (module 75's) — the *module-43's line: the stateless is the mobile/edge* (module 43's §1.1) — the *module-43's standing line: the stateful is the web default* (module 43's §1.1)).

The *problems*: (1) the *the `localStorage` is the XSS's* (module 43's §5): the *module-43's line: the `localStorage` is the XSS's* (module 43's §5) — the *module-19-02's XSS* (module 75's) — the *module-43's standing line: the `localStorage` is the XSS's* (module 43's §5).

(2) the *the stateless is the mobile/edge* (module 43's §1.1): the *module-43's line: the stateless is the mobile/edge* (module 43's §1.1) — the *the web's stateful is the default* (module 43's §1.1) — the *module-43's standing line: the stateful is the web default* (module 43's §1.1).

**Design**: the *the split* (the *the web's stateful* (module 43's §1.1) + the *the mobile's stateless* (module 43's §1.1) + the *the no `localStorage`* (module 43's §5) — the *module-43's line: the stateful is the web default* (module 43's §1.1) — the *module-43's standing line: the stateful is the web default + the stateless is the mobile/edge + the no `localStorage`* (module 43's §1.1 + module 43's §5)).

Produce: the *the split* (the *the web's stateful* (module 43's §1.1) + the *the mobile's stateless* (module 43's §1.1) + the *the no `localStorage`* (module 43's §5) — the *module-43's line: the stateful is the web default* (module 43's §1.1) — the *module-43's standing line: the stateful is the web default + the stateless is the mobile/edge + the no `localStorage`* (module 43's §1.1 + module 43's §5)).

<details>
<summary>Model answer</summary>
**The split** (module 43's §1.1 + module 43's §5):
1. **The web's stateful** (module 43's §1.1): the *the row's session* (module 43's §1.1) + the *the `HttpOnly` cookie* (module 45's) — the *module-43's line: the stateful is the web default* (module 43's §1.1).
2. **The mobile's stateless** (module 43's §1.1): the *the JWT* (module 43's §1.1) — the *the no `localStorage`* (module 43's §5) — the *the `keychain`/`keystore`* (module 43's §5) — the *module-43's line: the stateless is the mobile/edge* (module 43's §1.1).
**The generalization** (the *split's* pattern, the *module's* standing rule): **the *stateful is the web default* (module 43's §1.1) — the *the stateless is the mobile/edge* (module 43's §1.1) — the *the no `localStorage`* (module 43's §5) — the *module-43's standing line: the stateful is the web default + the stateless is the mobile/edge + the no `localStorage`* (module 43's §1.1 + module 43's §5)*.
</details>

## 10. Official Documentation

- The Next.js Authentication guide (the *the Data Access Layer*): https://nextjs.org/docs/app/guides/authentication
- The Next.js Data Security guide (the *the `taintUniqueValue`*): https://nextjs.org/docs/app/guides/data-security
- The OWASP Session Management Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Session_Management_Cheat_Sheet.html
- The module-44's Better Auth (the *the impl*): the module-44 (the phase-10's file-02)

## 11. What You Should Know Before Continuing

- [ ] I can state the *session as pointer* (module 1's: the cookie is the handle, the row is the truth) — the *no self-contained token in the cookie* (module 1's line)
- [ ] I can draw the *session lifecycle* (module 2's: the 5 verbs — the prove/remember/refresh/forget/revoke)
- [ ] I know the *session row's 7 columns* (module 3's) — the *the no `user_id` in the cookie* (module 3's line) — the *the no infinite session* (module 3's line)
- [ ] I know the *session read is the ONE function* (module 4's) — the *the no inline's cookie's read* (module 4's line)
- [ ] I know the *stateful is the web default* + the *stateless is the mobile/edge* (module 1.1's) — the *the no `localStorage` token* (module 5's line)
- [ ] I know the *401 is the who, the 403 is the what* (module 50's) — the *module-50's deep-dive* (module 50's)
- [ ] I've done the *lifecycle* (module 8's beginner) + the *session row* (module 8's intermediate) + the *revocation* (module 8's production) — the *artifacts* (module 20's)

**Next:** Module 44 — Better Auth (the *the architecture* — the *the setup* — the *the `drizzleAdapter`* — the *the `toNextJsHandler`* — the *the `nextCookies` plugin* — the *module-44's line: the Better Auth is the impl* (module 44's)).
