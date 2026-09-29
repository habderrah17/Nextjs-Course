# Module 77 — The Audit: Run the Checklist on the Capstone, Fix Every Finding

**Phase 19: Security · Module 77 of 101**

> **Where does this run?** The audit is a **process** (the checklist runs against the *running* capstone — module 77's §1); the *fixes* land where the threat lives (module 74's map — module 77's §1). The module-77's standing rule (module 74's model + module 75's attacks, now the audit level): **the audit is a *checklist run*, not a *review* (module 77's §1) — every finding has a *severity*, an *owner*, and a *fix date* (module 77's §1), and a finding without a fix is *not closed* (module 77's §1); the audit's output is the *artifact* (the `docs/security-audit.md`) — the *module-77's line: the audit is the finding's* (module 77's §1)** (module 77's §1).

---

## 1. Concept — The audit's 3 rules (the loop)

**The checklist's** (module 77's §1.1): the *the 10's categories* (module 77's §1.1) — the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *the no review's* (module 77's §1.1).

**The finding's** (module 77's §1.2): the *the severity's* + the *the owner's* + the *the fix date's* (module 77's §1.2) — the *module-77's line: the audit is the finding's* (module 77's §1.2) — the *the no closed's* (module 77's §1.2).

**The re-audit's** (module 77's §1.3): the *the no "it feels safe"* (module 77's §1.3) — the *module-77's line: the re-audit is the fix's* (module 77's §1.3) — the *module-69's line: the number is the change's* (module 69's §1).

## 2. Mental Model — The audit's loop (drawn)

```mermaid
flowchart TD
    A["THE CHECKLIST (module 77's §1.1) — the the 10's categories (module 77's §1.1) — the the no review's (module 77's §1.1)"] --> B["THE FINDINGS (module 77's §1.2) — the the severity's (module 77's §1.2) + the owner's (module 77's §1.2) + the fix date's (module 77's §1.2)"]
    B --> C["THE FIXES (module 77's §1.3) — the the no closed's (module 77's §1.2) — the the no "it feels safe"'s (module 77's §1.3)"]
    C --> D["THE RE-AUDIT (module 77's §1.3) — the the fix's (module 77's §1.3) — the the no "it feels safe"'s (module 77's §1.3)"]
    D --> A
```

**The audit's loop** (the module-77's mental model):
1. **The checklist** (module 77's §1.1): the *the 10's categories* — the *module-77's line: the audit is the checklist's* (module 77's §1.1).
2. **The finding** (module 77's §1.2): the *the severity's + the owner's + the fix date's* — the *module-77's line: the audit is the finding's* (module 77's §1.2).
3. **The re-audit** (module 77's §1.3): the *the fix's* — the *module-77's line: the re-audit is the fix's* (module 77's §1.3).

## 3. Architecture — The 10 categories (the checklist)

`FILE: docs/security-audit-checklist.md` (production pattern — the module-77's §3: the checklist's)

```md
## THE 10 CATEGORIES (module 77's §3 — the the checklist's (module 77's §3))

| # | Category (module 77's §3) | Check (module 77's §3) | Ref (module 77's §3) |
|---|---|---|---|
| 1 | Auth (module 77's §3.1) | The session's pointer (module 43's) — the the no localStorage's (module 43's §5) | Module 43 |
| 2 | Authz (module 77's §3.2) | The 2-gate's (module 48's) — the the 401→403's (module 48's) | Module 48 |
| 3 | Tenancy (module 77's §3.3) | The orgId's re-check (module 49's) — the the 404's no-leak (module 49's) | Module 49 |
| 4 | Input (module 77's §3.4) | The Zod's (module 53's) — the the no raw's (module 53's) | Module 53 |
| 5 | Output (module 77's §3.5) | The no `dangerouslySetInnerHTML`'s (module 75's §1.1) | Module 75 |
| 6 | Headers (module 77's §3.6) | The HSTS/nosniff/CSP's (module 76's §3.1 + §3.2) | Module 76 |
| 7 | Env (module 77's §3.7) | The no `NEXT_PUBLIC_`'s secret (module 76's §1) | Module 76 |
| 8 | Upload (module 77's §3.8) | The size/MIME/magic's (module 66's) — the the no `public/`'s (module 66's) | Module 66 |
| 9 | Cache (module 77's §3.9) | The no session's in the cache (module 22's) — the the cacheTag's (module 20's) | Module 22 |
| 10 | Monitoring (module 77's §3.10) | The no-log's PII (module 69's §3.2) — the the alert's (module 77's §3.10) | Module 69 |

/* THE RULE (module 77's §3): the the checklist's (module 77's §3) — the the 10's categories (module 77's §3) — the the no review's (module 77's §1.1) */
```

## 4. Production Code — The capstone's audit (module 77's §4)

`FILE: docs/security-audit.md` (production pattern — the module-77's §4: the findings')

```md
## THE CAPSTONE'S AUDIT (module 77's §4 — the the findings' (module 77's §4))

| # | Category | Finding (module 77's §4) | Severity | Owner | Fix | Status |
|---|---|---|---|---|---|---|
| F-01 | Env | The `S3_SECRET_ACCESS_KEY` is in `NEXT_PUBLIC_` | **Critical** | The infra | The no `NEXT_PUBLIC_` (module 76's §1) | ✅ |
| F-02 | Auth | The `localStorage`'s token (module 43's §5) | **Critical** | The auth | The session's pointer (module 43's) | ✅ |
| F-03 | Authz | The no 401→403's (module 48's) | **High** | The auth | The 2-gate's (module 48's) | ✅ |
| F-04 | Tenancy | The no orgId's re-check (module 49's) | **Critical** | The data | The orgId's re-check (module 49's) | ✅ |
| F-05 | Output | The `dangerouslySetInnerHTML`'s (module 75's §1.1) | **High** | The UI | The text's (module 75's §3.1) | ✅ |
| F-06 | Headers | The no CSP (module 76's §1.3) | **Medium** | The edge | The report-only's (module 76's §3.2) | ✅ |
| F-07 | Env | The no env's validator (module 76's §3.3) | **Medium** | The infra | The Zod's (module 76's §3.3) | ✅ |
| F-08 | Upload | The no magic's bytes (module 66's) | **High** | The upload | The magic's (module 66's) | ✅ |
| F-09 | Cache | The session's in the cache (module 22's) | **Critical** | The cache | The no session's (module 22's) | ✅ |
| F-10 | Monitoring | The no-log's PII (module 69's §3.2) | **Medium** | The ops | The no-log's (module 69's §3.2) | ✅ |

/* THE RULE (module 77's §4): the the finding's (module 77's §1.2) — the the severity's (module 77's §1.2) — the the fix's (module 77's §1.3) — the the no closed's (module 77's §1.2) */
```

**The module-77's line:** the *finding's* (module 77's §1.2) — the *severity's* (module 77's §1.2) — the *fix's* (module 77's §1.3) — the *no closed's* (module 77's §1.2).

## 5. Common Mistakes (the audit's failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The review's** (module 77's §1.1's line violated) | the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *the review's is the *no's* (module 77's §1.1) — the *module-77's line: the no review's* (module 77's §1.1) — the *no review's* (module 77's §1.1)* | the *the checklist's (module 77's §3) — the *module-77's line: the audit is the checklist's* (module 77's §1.1)* |
| **The no severity** (module 77's §1.2's line violated) | the *module-77's line: the finding's* (module 77's §1.2) — the *the no severity's is the *no's* (module 77's §1.2) — the *module-77's line: the no severity* (module 77's §1.2) — the *no severity* (module 77's §1.2)* | the *the severity's (module 77's §1.2) — the *module-77's line: the finding's* (module 77's §1.2)* |
| **The no owner** (module 77's §1.2's line violated) | the *module-77's line: the finding's* (module 77's §1.2) — the *the no owner's is the *no's* (module 77's §1.2) — the *module-77's line: the no owner* (module 77's §1.2) — the *no owner* (module 77's §1.2)* | the *the owner's (module 77's §1.2) — the *module-77's line: the finding's* (module 77's §1.2)* |
| **The no fix date** (module 77's §1.2's line violated) | the *module-77's line: the finding's* (module 77's §1.2) — the *the no fix date's is the *no's* (module 77's §1.2) — the *module-77's line: the no fix date* (module 77's §1.2) — the *no fix date* (module 77's §1.2)* | the *the fix date's (module 77's §1.2) — the *module-77's line: the finding's* (module 77's §1.2)* |
| **The "it feels safe"** (module 77's §1.3's line violated) | the *module-77's line: the re-audit is the fix's* (module 77's §1.3) — the *the "it feels safe"'s is the *no's* (module 77's §1.3) — the *module-77's line: the no "it feels safe"* (module 77's §1.3) — the *no "it feels safe"* (module 77's §1.3)* | the *the re-audit's (module 77's §1.3) — the *module-77's line: the re-audit is the fix's* (module 77's §1.3)* |
| **The no re-audit** (module 77's §1.3's line violated) | the *module-77's line: the re-audit is the fix's* (module 77's §1.3) — the *the no re-audit's is the *no's* (module 77's §1.3) — the *module-77's line: the no re-audit* (module 77's §1.3) — the *no re-audit* (module 77's §1.3)* | the *the re-audit's (module 77's §1.3) — the *module-77's line: the re-audit is the fix's* (module 77's §1.3)* |

## 6. Security Notes

- **The no closed's** (module 77's §1.2): the *module-77's line: the finding without a fix is not closed* (module 77's §1.2) — the *module-74's* *deep-dive* (module 74's).
- **The severity's** (module 77's §1.2): the *module-77's line: the Critical's is the no-deploy's* (module 77's §1.2) — the *module-74's* *deep-dive* (module 74's).
- **The no review's** (module 77's §1.1): the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *module-75's* *deep-dive* (module 75's).

## 7. Performance Notes

- **The no review's** (module 77's §1.1): the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *the no meeting's cost* (module 77's §1.1).
- **The re-audit's** (module 77's §1.3): the *module-77's line: the re-audit is the fix's* (module 77's §1.3) — the *the no "it feels safe"'s cost* (module 77's §1.3).
- **The 10's** (module 77's §3): the *module-77's line: the checklist's* (module 77's §3) — the *the 10's categories' cost* (module 77's §3).

## 8. Exercise

**Beginner.** *The checklist's* (module 77's §3): the *the 10's categories* (module 3's) + the *the check's* (module 3's) — *build the table* — the *artifact: the checklist's* (module 3's).

**Intermediate.** *The finding's* (module 77's §4): the *the severity's* (module 4's) + the *the owner's* (module 4's) + the *the fix date's* (module 4's) — *build the findings' table* — the *artifact: the findings'* (module 4's).

**Production.** *The re-audit's* (module 77's §1.3): the *the fix's* (module 4's) + the *the re-audit's* (module 1.3's) — *run it* — the *artifact: the re-audit's* (module 1.3's).

## 9. Architecture Challenge

**Prompt:** The *"the team's 'audit' was a meeting where the CTO said it looks fine — no checklist, no findings, no fix dates"* (the *module-77's* *audit* — the *module-74's* *model* — the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *module-74's line: the app is the boundary's* (module 74's §1.3) — the *module-77's standing line: the audit is the checklist's + the finding's + the re-audit's* (module 77's §1.1 + module 77's §1.2 + module 77's §1.3)).

The *problems*: (1) the *the no checklist's* (the *the no audit's* (module 77's §1.1) — the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *module-77's standing line: the audit is the checklist's* (module 77's §1.1)).

(2) the *the no finding's* (the *the no severity's* (module 77's §1.2) — the *module-77's line: the audit is the finding's* (module 77's §1.2) — the *module-77's standing line: the finding's* (module 77's §1.2)).

**Design**: the *the audit's remediation* (the *the checklist's* (module 77's §3) + the *the finding's* (module 77's §4) + the *the re-audit's* (module 77's §1.3) — the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *module-77's standing line: the audit is the checklist's + the finding's + the re-audit's* (module 77's §1.1 + module 77's §1.2 + module 77's §1.3)).

Produce: the *the audit's remediation* (the *the checklist's* (module 77's §3) + the *the finding's* (module 77's §4) + the *the re-audit's* (module 77's §1.3) — the *module-77's line: the audit is the checklist's* (module 77's §1.1) — the *module-77's standing line: the audit is the checklist's + the finding's + the re-audit's* (module 77's §1.1 + module 77's §1.2 + module 77's §1.3)).

<details>
<summary>Model answer</summary>
**The audit's remediation** (module 77's §3 + module 77's §4 + module 77's §1.3):
1. **The checklist's** (module 77's §3): the *the 10's categories replace the meeting's* — the *module-77's line: the audit is the checklist's* (module 77's §1.1).
2. **The finding's** (module 77's §4): the *the severity's + the owner's + the fix date's* — the *module-77's line: the audit is the finding's* (module 77's §1.2).
3. **The re-audit's** (module 77's §1.3): the *the fix's + the re-audit's* — the *module-77's line: the re-audit is the fix's* (module 77's §1.3).
**The generalization** (the *audit's* pattern, the *module's* standing rule): **the *audit is the checklist's* (module 77's §1.1) — the *the finding's* (module 77's §1.2) — the *the re-audit is the fix's* (module 77's §1.3) — the *module-77's standing line: the audit is the checklist's + the finding's + the re-audit's* (module 77's §1.1 + module 77's §1.2 + module 77's §1.3)*.
</details>

## 10. Official Documentation

- OWASP Top 10: https://owasp.org/Top10/
- OWASP Testing Guide: https://owasp.org/www-project-web-security-testing-guide/
- Next.js: Security: https://nextjs.org/docs/app/guides/security
- The module-74's model: the module-74 (the phase-19's file-01)
- The module-75's attack: the module-75 (the phase-19's file-02)
- The module-76's headers: the module-76 (the phase-19's file-03)

## 11. What You Should Know Before Continuing

- [ ] I can state the *3 rules* (module 1's: the checklist/finding/re-audit) — the *module-77's line: the audit is the finding's* (module 1's)
- [ ] I know the *checklist's* (module 1.1's) — the *the 10's categories* (module 3's)
- [ ] I know the *finding's* (module 1.2's) — the *the severity's + the owner's + the fix date's* (module 4's)
- [ ] I know the *re-audit's* (module 1.3's) — the *the no "it feels safe"* (module 4's)
- [ ] I've done the *checklist's* (module 8's beginner) + the *finding's* (module 8's intermediate) + the *re-audit's* (module 8's production) — the *artifacts* (module 20's)

**Phase 19 complete.** Security — the model (module 74's), the attacks (module 75's), the headers/env (module 76's), the audit (module 77's).

**Next:** Module 78 — Phase 20 (the *the observability's* — the *module-78's line: the log is the structured's* (module 78's)).
