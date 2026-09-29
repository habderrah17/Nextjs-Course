# Module 95 — The Staged Build: 15 Stages, Each with a Folder Tree, a Data Flow, and a Boundary Audit

**Phase 24: Capstone · Module 95 of 101**

> **Where does this run?** The build is **code** (the capstone's, module 95's §1); the *audit* is a **checklist** (the 8's, module 92's §3). The module-95's standing rule (module 92's review rule, now the build level): **the build is the *staged's* (module 95's §1) — 15 stages, each *shippable* (module 95's §1); every stage has a *folder tree* (the delta, module 95's §1), a *data flow* (the arrow, module 95's §1), and a *boundary audit* (the 8's checks, module 92's §3) (module 95's §1); and a stage that *fails the audit* is *not done* (module 95's §1)** (module 95's §1).

---

## 1. Concept — The 3 per stage (the map)

**The tree's** (module 95's §1.1): the *the delta's* (module 95's §1.1) — the *module-95's line: the tree is the delta's* (module 95's §1.1) — the *module-94's* *folder* (module 94's §1.1).

**The flow's** (module 95's §1.2): the *the arrow's* (module 95's §1.2) — the *module-95's line: the flow is the arrow's* (module 95's §1.2) — the *module-4's* *fetch* (module 4's).

**The audit's** (module 95's §1.3): the *the 8's checks* (module 92's §3) — the *module-95's line: the audit is the 8's* (module 92's §3) — the *module-92's* *checklist* (module 92's §3).

## 2. Mental Model — The 15 stages (drawn)

```mermaid
flowchart LR
    S1["1: Scaffold"] --> S2["2: Auth"]
    S2 --> S3["3: Org + RBAC"]
    S3 --> S4["4: Catalog CRUD"]
    S4 --> S5["5: 2FA"]
    S5 --> S6["6: Search/Filter"]
    S6 --> S7["7: Data Screens"]
    S7 --> S8["8: Orders"]
    S8 --> S9["9: Admin Console"]
    S9 --> S10["10: Dashboard"]
    S10 --> S11["11: i18n"]
    S11 --> S12["12: SEO"]
    S12 --> S13["13: Tests"]
    S13 --> S14["14: Perf Pass"]
    S14 --> S15["15: CI/CD"]
```

## 3. Architecture — The 15 stages (the table)

`FILE: docs/staged-build.md` (production pattern — the module-95's §3: the table's)

```md
## THE 15 STAGES (module 95's §3 — the the table's (module 95's §3))

| # | Stage (module 95's §3) | The delta's tree (module 95's §3) | The flow's (module 95's §3) | The ref's (module 95's §3) |
|---|---|---|---|---|
| 1 | The scaffold's (module 95's §3.1) | The `src/app`'s + the `src/components`'s + the `src/lib` (module 95's §3.1) | The no flow's (module 95's §3.1) | Module 55 |
| 2 | The auth's (module 95's §3.2) | The `src/app/(auth)`'s + the `src/lib/auth` (module 95's §3.2) | The login's → the session's (module 95's §3.2) | Module 43 |
| 3 | The org's + the RBAC's (module 95's §3.3) | The `src/app/(org)`'s + the `src/services/members` (module 95's §3.3) | The invite's → the role's (module 95's §3.3) | Module 48 |
| 4 | The catalog's CRUD's (module 95's §3.4) | The `src/services/products`'s + the `src/db/schema` (module 95's §3.4) | The product's → the DB's (module 95's §3.4) | Module 5 |
| 5 | The 2FA's (module 95's §3.5) | The `src/app/(auth)/2fa`'s + the `twoFactor`'s (module 95's §3.5) | The code's → the session's (module 95's §3.5) | Module 44 |
| 6 | The search's + the filter's (module 95's §3.6) | The `src/components/filters`'s (module 95's §3.6) | The URL's → the DB's WHERE (module 95's §3.6) | Module 67 |
| 7 | The data's screens' (module 95's §3.7) | The `src/app/(org)/products`'s (module 95's §3.7) | The list's → the detail's (module 95's §3.7) | Module 67 |
| 8 | The orders's (module 95's §3.8) | The `src/services/orders`'s + the `checkout`'s (module 95's §3.8) | The cart's → the PSP's (module 95's §3.8) | Module 54 |
| 9 | The admin's console's (module 95's §3.9) | The `src/app/(org)/admin`'s (module 95's §3.9) | The admin's → the RBAC's (module 95's §3.9) | Module 48 |
| 10 | The dashboard's (module 95's §3.10) | The `src/app/(org)/dashboard`'s + the chart's island (module 95's §3.10) | The stats's → the chart's (module 95's §3.10) | Module 68 |
| 11 | The i18n's (module 95's §3.11) | The `src/i18n`'s + the `messages/` (module 95's §3.11) | The locale's → the string's (module 95's §3.11) | Module 88 |
| 12 | The SEO's (module 95's §3.12) | The `src/app/(public)`'s + the `sitemap.ts`'s (module 95's §3.12) | The metadata's → the OG's (module 95's §3.12) | Module 61 |
| 13 | The tests's (module 95's §3.13) | The `e2e/`'s + the `*.test.ts`'s (module 95's §3.13) | The pyramid's (module 95's §3.13) | Module 78 |
| 14 | The perf's pass' (module 95's §3.14) | The no delta's (module 95's §3.14) | The before's → the after's (module 95's §3.14) | Module 69 |
| 15 | The CI/CD's (module 95's §3.15) | The `.github/workflows`'s + the `Dockerfile`'s (module 95's §3.15) | The pipeline's (module 95's §3.15) | Module 86 |

/* THE RULE (module 95's §3): the the stage's (module 95's §3) — the the delta's (module 95's §1.1) — the the flow's (module 95's §1.2) — the the audit's (module 92's §3) */
```

## 4. Production Code — The reference tree (module 95's §4)

`FILE: docs/capstone-tree.md` (production pattern — the module-95's §4: the tree's)

```
src/
  app/
    (auth)/            # stage 2 — the public's auth (module 95's §3.2)
      login/ register/ 2fa/
    (org)/             # stage 3 — the tenant's app (module 95's §3.3)
      products/ orders/ dashboard/ admin/
    (public)/          # stage 12 — the shop's (module 95's §3.12)
      [slug]/
    api/
      v1/              # stage 8 — the BFF's (module 95's §3.8)
      track/           # stage 13 — the beacon's (module 95's §3.13)
      metrics/         # stage 14 — the vitals' (module 95's §3.14)
  components/          # stage 1 — the UI's (module 95's §3.1)
  lib/                 # stage 1 — the no domain's (module 95's §3.1)
  services/            # stage 4 — the domain's (module 95's §3.4)
  db/                  # stage 4 — the schema's (module 95's §3.4)
  stores/              # stage 10 — the Zustand's (module 95's §3.10)
  i18n/                # stage 11 — the locale's (module 95's §3.11)
messages/              # stage 11 — the string's (module 95's §3.11)
e2e/                   # stage 13 — the journey's (module 95's §3.13)
```

**The module-95's line:** the *delta's* (module 95's §1.1) — the *tree's* (module 95's §4) — the *no domain's in the `lib`'s* (module 95's §3.1).

## 5. Common Mistakes (the build's failures)

| Mistake | The symptom | Fix |
|---|---|---|
| **The no stage** (module 95's §1's line violated) | the *module-95's line: the build is the staged's* (module 95's §1) — the *the no stage's is the *no's* (module 95's §1) — the *module-95's line: the no stage* (module 95's §1) — the *no stage* (module 95's §1)* | the *the 15's stages (module 95's §3) — the *module-95's line: the build is the staged's* (module 95's §1)* |
| **The no audit** (module 95's §1.3's line violated) | the *module-95's line: the audit is the 8's* (module 92's §3) — the *the no audit's is the *no's* (module 92's §3) — the *module-95's line: the no audit* (module 92's §3) — the *no audit* (module 92's §3)* | the *the 8's checks (module 92's §3) — the *module-95's line: the audit is the 8's* (module 92's §3)* |
| **The no flow** (module 95's §1.2's line violated) | the *module-95's line: the flow is the arrow's* (module 95's §1.2) — the *the no flow's is the *no's* (module 95's §1.2) — the *module-95's line: the no flow* (module 95's §1.2) — the *no flow* (module 95's §1.2)* | the *the flow's (module 95's §3) — the *module-95's line: the flow is the arrow's* (module 95's §1.2)* |
| **The no delta** (module 95's §1.1's line violated) | the *module-95's line: the tree is the delta's* (module 95's §1.1) — the *the no delta's is the *no's* (module 95's §1.1) — the *module-95's line: the no delta* (module 95's §1.1) — the *no delta* (module 95's §1.1)* | the *the delta's (module 95's §3) — the *module-95's line: the tree is the delta's* (module 95's §1.1)* |
| **The shippable's no** (module 95's §1's line violated) | the *module-95's line: the shippable's* (module 95's §1) — the *the shippable's no's is the *no's* (module 95's §1) — the *module-95's line: the no shippable* (module 95's §1) — the *no shippable* (module 95's §1)* | the *the stage's (module 95's §3) — the *module-95's line: the shippable's* (module 95's §1)* |
| **The failed's audit** (module 95's §1's line violated) | the *module-95's line: the no failed's audit* (module 95's §1) — the *the failed's audit's is the *no's* (module 95's §1) — the *module-95's line: the no failed's audit* (module 95's §1) — the *no failed's audit* (module 95's §1)* | the *the audit's (module 92's §3) — the *module-95's line: the no failed's audit* (module 95's §1)* |

## 6. Security Notes

- **The tenancy's** (module 37's): the *module-95's line: the audit is the 8's* (module 92's §3) — the *module-37's* *deep-dive* (module 37's).
- **The RBAC's** (module 48's): the *module-95's line: the audit is the 8's* (module 92's §3) — the *module-48's* *deep-dive* (module 48's).
- **The PSP's** (module 54's): the *module-95's line: the flow is the arrow's* (module 95's §1.2) — the *module-54's* *deep-dive* (module 54's).

## 7. Performance Notes

- **The before's** (module 95's §3.14): the *module-95's line: the perf's pass'* (module 95's §3.14) — the *the before's numbers* (module 69's §1).
- **The after's** (module 95's §3.14): the *module-95's line: the perf's pass'* (module 95's §3.14) — the *the after's numbers* (module 69's §1).
- **The no delta's** (module 95's §3.14): the *module-95's line: the perf's pass'* (module 95's §3.14) — the *the no tree's delta* (module 95's §3.14).

## 8. Exercise

**Beginner.** *The stages 1–5's* (module 95's §3.1–§3.5): the *the 5's deltas* (module 3's) + the *the 5's flows* (module 3's) — *build them* — the *artifact: the 5's stages* (module 3's).

**Intermediate.** *The stages 6–10's* (module 95's §3.6–§3.10): the *the 5's deltas* (module 3's) + the *the 5's flows* (module 3's) — *build them* — the *artifact: the 5's stages* (module 3's).

**Production.** *The stages 11–15's* (module 95's §3.11–§3.15): the *the 5's deltas* (module 3's) + the *the 5's audits* (module 3's) — *build them* — the *artifact: the 5's stages* (module 3's).

## 9. Architecture Challenge

**Prompt:** The *"the team builds the capstone in 3 weeks with no stages — no folder tree per stage, no data flow, no audit, and the perf pass is skipped"* (the *module-95's* *build* — the *module-92's* *audit* — the *module-95's line: the build is the staged's* (module 95's §1) — the *module-92's line: the audit is the 8's* (module 92's §3) — the *module-95's standing line: the 15's stages + the delta's + the flow's + the audit's* (module 95's §1 + module 95's §1.1 + module 95's §1.2 + module 92's §3)).

The *problems*: (1) the *the no stage* (the *the no shippable's* (module 95's §1) — the *module-95's line: the build is the staged's* (module 95's §1) — the *module-95's standing line: the 15's stages* (module 95's §1)).

(2) the *the no audit* (the *the no 8's* (module 92's §3) — the *module-95's line: the audit is the 8's* (module 92's §3) — the *module-95's standing line: the audit's* (module 92's §3)).

**Design**: the *the build's remediation* (the *the 15's stages* (module 3's) + the *the delta's per stage* (module 3's) + the *the 8's audits* (module 3's) — the *module-95's line: the build is the staged's* (module 95's §1) — the *module-95's standing line: the 15's stages + the delta's + the flow's + the audit's* (module 95's §1 + module 95's §1.1 + module 95's §1.2 + module 92's §3)).

Produce: the *the build's remediation* (the *the 15's stages* (module 3's) + the *the delta's per stage* (module 3's) + the *the 8's audits* (module 3's) — the *module-95's line: the build is the staged's* (module 95's §1) — the *module-95's standing line: the 15's stages + the delta's + the flow's + the audit's* (module 95's §1 + module 95's §1.1 + module 95's §1.2 + module 92's §3)).

<details>
<summary>Model answer</summary>
**The build's remediation** (module 95's §3 + module 95's §4 + module 92's §3):
1. **The 15's stages** (module 95's §1): the *the 3's weeks become the 15's stages'* — the *module-95's line: the build is the staged's* (module 95's §1).
2. **The delta's per stage** (module 95's §1.1): the *the no-tree's becomes the delta's* — the *module-95's line: the tree is the delta's* (module 95's §1.1).
3. **The 8's audits** (module 92's §3): the *the no-audit's becomes the 8's* — the *module-95's line: the audit is the 8's* (module 92's §3).
**The generalization** (the *build's* pattern, the *module's* standing rule): **the *15's stages* (module 95's §1) — the *the delta's* (module 95's §1.1) — the *the flow's* (module 95's §1.2) — the *the audit's* (module 92's §3) — the *module-95's standing line: the 15's stages + the delta's + the flow's + the audit's* (module 95's §1 + module 95's §1.1 + module 95's §1.2 + module 92's §3)*.
</details>

## 10. Official Documentation

- Next.js: Project Structure: https://nextjs.org/docs/app/getting-started/project-structure
- The module-92's audit: the module-92 (the phase-23's file-04)
- The module-96's review: the module-96 (the phase-24's file-04)

## 11. What You Should Know Before Continuing

- [ ] I can state the *15 stages* (module 2's: the 1's–15's) — the *module-95's line: the build is the staged's* (module 2's)
- [ ] I know the *tree is the delta's* (module 1.1's) — the *the delta's per stage* (module 3's)
- [ ] I know the *flow is the arrow's* (module 1.2's) — the *the flow's per stage* (module 3's)
- [ ] I know the *audit is the 8's* (module 1.3's) — the *the 8's checks per stage* (module 92's §3)
- [ ] I know the *shippable's* (module 1's) — the *the no failed's audit* (module 1's)
- [ ] I've done the *1–5's* (module 8's beginner) + the *6–10's* (module 8's intermediate) + the *11–15's* (module 8's production) — the *artifacts* (module 20's)

**Next:** Module 96 — The Senior Architect Review (the *the review's* — the *module-96's line: the review is the architect's* (module 96's)).
