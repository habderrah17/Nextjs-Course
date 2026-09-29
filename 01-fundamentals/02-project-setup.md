# Module 02 — Project Setup: `create-next-app`, Every Option Explained

**Phase 1: Foundations · Module 2 of 101**

> **Where does the code in this module run?** Tooling runs on your machine (build time). The output — the app — is the five-surface architecture from Module 01. This module's job: produce a scaffold where *every default was a decision you made consciously.*

---

## 1. Concept — Why the official scaffolder, and why "explain every option"?

`create-next-app` is the official recommended way to start a Next.js project (it is literally the first command in the official docs' Quick Start). It generates: the folder structure, TypeScript config, lint config, Tailwind (if chosen), the `app/` directory, package.json scripts, and — as of 16.x — an `AGENTS.md` file for AI coding agents.

"Explain every option" matters because the CLI's *default* choices encode an architecture: which linter = your code-health model; `src/` or not = your project shape; React Compiler on/off = your client-performance posture; Turbopack = your build pipeline. Accepting defaults blindly is how teams inherit decisions they can't explain.

---

## 2. Mental Model — the project as four layers

A Next.js project is not "a bunch of files." It is four layers, each with a distinct lifecycle:

```
L1 CONFIG      next.config.ts, tsconfig.json, eslint config, package.json
               → read at build/start; changes require re-build
L2 ROUTES      app/ (or src/app/)  → file-system = routing table
L3 CODE        components/, services/, lib/, actions/, schemas/, …
L4 ASSETS      public/ (served as-is from the web root), images, fonts (next/font)
```

**Rule L1:** if it affects the *framework's behavior* (caching, bundling, rewrites, image domains) it belongs in `next.config.ts`, not in code.
**Rule L2:** if a URL exists, a file exists. Routing is *derived*, never configured imperatively (rewrites/redirects in config are the narrow exception — module 02-09).
**Rule L3:** code is organized by *boundary and domain*, not by file type (module 03 of this phase).
**Rule L4:** `public/` files are immutable in practice (they bypass optimization and are served with strong caching) — dynamic assets go through `next/image` or Route Handlers.

---

## 3. Architecture — what `next dev` / `next build` / `next start` actually do

```mermaid
flowchart LR
    subgraph DEV["next dev [Turbopack — default]"]
        D1[on-demand compilation per request]
        D2[Fast Refresh — sub-200ms HMR]
        D3[dev overlay: error + caching insights<br/>(e.g. 'wrap in Suspense' fix cards)]
        D4[dev logs: Compile vs Render timing]
    end
    subgraph BUILD["next build [Turbopack — default]"]
        B1[TypeScript check]
        B2[compile + bundle (client & server)]
        B3[prerender: static pages + static shell<br/>(cacheComponents) with dynamic holes]
        B4[fonts, images, metadata generation]
        B5[.next/ output]
    end
    subgraph START["next start"]
        S1[Node 24 process serving .next]
        S2[CDN in front in production]
    end
    DEV -->|same code| BUILD --> START
```

Facts to internalize (all verified against the official docs):

- **Turbopack is the default bundler** for `next dev` and `next build` in Next.js 16. Webpack remains available via `--webpack` (opt-out, for migration only). `next build` *fails* if a webpack config is detected.
- **Dev server logs now break time into `Compile` (routing + compilation) and `Render` (your code + React)** — your first performance instrument, free (module 18).
- **`next build` output** lists each step with its duration (Collecting page data, Generating static pages, …) — read it every build (module 18-01).
- **Minimum Node.js: 20.9.** We target **Node 24 LTS**.
- **Minimum TypeScript: 5.1** (we use the latest 5.x).
- **`next lint` is removed in 16** — the `lint` script runs `eslint` directly (flat config).

---

## 4. Examples — the commands and every prompt

### 4.1 Scaffolding

```bash
# Use the package manager you use (pnpm shown):
pnpm create next-app@latest commerce-ops

# Non-interactive (CI, or you've decided everything):
pnpm create next-app@latest commerce-ops --yes
```

`--yes` skips prompts using saved preferences or defaults. The **default setup enables: TypeScript, Tailwind CSS, ESLint, App Router, and Turbopack**, with import alias `@/*`, and includes `AGENTS.md` (with a `CLAUDE.md` referencing it) to guide coding agents to write up-to-date Next.js code.

### 4.2 The interactive prompts (choose **customize settings** in this course)

You will see (verified prompt list, 16.x):

```
Would you like to use TypeScript?                     → Yes
Which linter would you like to use?                   → ESLint   (see 4.3)
Would you like to use React Compiler?                 → Yes     (see 4.4)
Would you like to use Tailwind CSS?                   → Yes     (see 4.5)
Would you like your code inside a src/ directory?     → Yes     (see 4.6)
Would you like to use App Router? (recommended)       → Yes
Would you like to customize the import alias (@/*)?   → No (keep @/*)
Would you like to include AGENTS.md…?                 → Yes     (see 4.7)
```

### 4.3 Linter: ESLint vs Biome vs None — decide, don't default

| | **ESLint 9 (flat config)** ✅ chosen | **Biome** | None |
|---|---|---|---|
| Ecosystem | Largest; every plugin (react-hooks **with React Compiler lint rules**, security plugins) | One binary, extremely fast, less plugin depth | — |
| React Compiler rules | Available via `eslint-plugin-react-hooks` recommended presets | Not the primary path | Lost |
| Course decision | **Primary** — the course relies on React Compiler lint rules and plugin extensibility | Fully valid alternative (module 25-03 tradeoffs) | Never for production code |

`FILE: package.json` (generated) — the scripts that matter:

```json
{
  "scripts": {
    "dev": "next dev",
    "build": "next build",
    "start": "next start",
    "lint": "eslint",
    "lint:fix": "eslint --fix"
  }
}
```

Note: no `next lint` — that command no longer exists in 16.x. (Old tutorials: see module 25-04 changelog.)

### 4.4 React Compiler — yes, with eyes open

React Compiler 1.0 (stable, Oct 2025) automatically memoizes components — replacing most manual `useMemo`/`useCallback`. In Next.js 16 it has **stable support**; the CLI prompt sets it up for you.

`FILE: next.config.ts` (generated when you accept)

```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  reactCompiler: true,
}

export default nextConfig
```

**Tradeoff (honest):** it changes re-render behavior automatically (usually better), but a non-trivial fraction of legacy components have latent bugs it surfaces (e.g., components that accidentally depended on re-rendering). The compiler's lint rules flag un-optimizable components so you fix them *knowing why*. We keep it on and learn the diagnostics — module 18-03 covers it in depth.

### 4.5 Tailwind CSS v4

The CLI sets up **Tailwind v4**: CSS-first configuration (no `tailwind.config.js` by default), `@import "tailwindcss"` in `globals.css`, automatic content detection via `@source`, OKLCH-based defaults. Module 13-01 goes deep; here you only need to know: if a tutorial shows `tailwind.config.ts` with a `content` array, it's pre-v4.

### 4.6 The `src/` directory — yes, with the reason

With `src/`, framework files (`app/`, `proxy.ts`, `middleware.ts`) live in `src/` alongside *your* code, keeping root-level config (`package.json`, `next.config.ts`, `tsconfig.json`) visually separate from source. It also matches the convention in most production repos and makes `@/*` aliasing unambiguous. Cost: none functionally. **We use it.**

### 4.7 `AGENTS.md` — the 16.x addition

`AGENTS.md` (and a `CLAUDE.md` referencing it) tells AI coding assistants the project's conventions and points them at current docs — because LLM training data is older than the framework. If your team uses AI agents (most do), keep it updated when you change conventions.

---

## 5. Production Code — the post-scaffold checklist

Run this sequence after scaffolding. Each step exists because something breaks without it.

### 5.1 Enable Cache Components (the current caching model)

The scaffolder does **not** enable this for you. We enable it, because the Cache Components model is the current documented default path (the old model is literally titled "Caching (Previous Model)" in the docs). Module 05 teaches the full model; enabling it now means the whole course runs on one consistent caching story.

`FILE: next.config.ts` (production pattern)

```ts
import type { NextConfig } from 'next'

const nextConfig: NextConfig = {
  reactCompiler: true,
  cacheComponents: true, // Next.js 16: 'use cache' + cacheLife + cacheTag model
}

export default nextConfig
```

> Good-to-know (official): with Cache Components enabled, `GET` Route Handlers follow the same prerendering model as pages. The flag is opt-in and can be removed; the previous model keeps working (module 05-07 covers both and the migration).

### 5.2 TypeScript strictness

`FILE: tsconfig.json` — verify the scaffold has (add if missing):

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "paths": { "@/*": ["./src/*"] }
  }
}
```

`noUncheckedIndexedAccess` is not on by default in the scaffold — we add it: it turns `array[i]` into `T | undefined`, which is exactly right for data that may or may not exist (route params, list indexes). It is a small config line that catches a whole class of production bugs.

### 5.3 Environment variables — the four files

Next.js loads env files in a documented order (later wins). Set up the split from day one (module 19-03 goes deep):

```
.env                # committed; defaults for all envs (no secrets)
.env.local          # local overrides; NEVER committed (.gitignore'd)
.env.development    # development-only values
.env.production     # production-only values (secrets live in your deploy platform, not here)
```

`FILE: .env` (committed, example values)

```dotenv
# Public by definition — inlined into the client bundle.
NEXT_PUBLIC_APP_URL=http://localhost:3000
# NEXT_PUBLIC_ vars only. No database URLs, no API keys.
```

`FILE: .env.local` (gitignored)

```dotenv
# Server-only. Never referenced from any 'use client' file.
DATABASE_URL=postgresql://postgres:postgres@localhost:5432/commerce_ops
BETTER_AUTH_SECRET=change-me-32-chars-min
```

`.gitignore` already contains `.env*.local` — **verify it**, and extend if you add `.env.development.local` etc. Add to your pre-commit/CI: a grep for `NEXT_PUBLIC_` in `.env.local` should fail loudly (a `NEXT_PUBLIC_` var set only in `.env.local` is a build-time landmine in CI — module 19-03).

### 5.4 CI skeleton (the five gates)

`FILE: .github/workflows/ci.yml` (production pattern)

```yaml
name: CI
on: [push, pull_request]

jobs:
  quality:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 24
          cache: pnpm
      - uses: pnpm/action-setup@v4
      - run: pnpm install --frozen-lockfile
      - run: pnpm lint
      - run: pnpm exec tsc --noEmit
      - run: node scripts/audit-boundary.mjs   # from Module 01
      - run: pnpm test
      - run: pnpm build
```

The order is the pedagogy: **fast → slow, cheap → expensive**. A type error should never wait behind a 3-minute build.

### 5.5 The finished scaffold tree (target state after this module)

```
commerce-ops/
├── .github/workflows/ci.yml
├── .env                    # committed defaults (public vars only)
├── .env.local              # gitignored secrets
├── AGENTS.md
├── eslint.config.mjs
├── next.config.ts          # reactCompiler + cacheComponents
├── package.json
├── pnpm-lock.yaml
├── tsconfig.json           # strict + noUncheckedIndexedAccess
└── src/
    ├── app/
    │   ├── layout.tsx
    │   ├── page.tsx
    │   ├── globals.css
    │   └── favicon.ico
    └── proxy.ts            # created in module 02-05 area (empty shell now, filled later)
```

(We will *not* create `src/services`, `src/db`, etc. yet — module 03 of this phase adds them with reasons. Folders are born with justification, never as decoration.)

---

## 6. Common Mistakes

| Mistake | Consequence | Fix |
|---|---|---|
| `npm create next-app` with old `next` global cache | You scaffold 15.x/14.x | Always `@latest`; verify the banner prints "Next.js 16" |
| Accepting defaults without knowing them | You can't explain your own build pipeline | This module; answer: "what does each of my 8 answers do?" |
| `.env.local` values referenced in code but missing in CI | Build works locally, breaks in CI (or worse: silently different behavior) | Every var in code must have a committed default or be required-and-failed-fast at boot (module 19-03) |
| Committing `.env.local` | **Secrets in git history** | Verify `.gitignore`; if it happened: rotate the secret (git history is public) |
| Adding `webpack.config.js` in Next 16 | `next build` fails (Turbopack default) | Don't. If you truly need webpack: `--webpack` opt-out, documented as a migration crutch |
| Putting `process.env.DATABASE_URL` in a component to "see if it's there" | It ships the var *name* (fine) but trains the habit of touching `process.env` in render code | Env access belongs in one module: `src/lib/env.ts` (module 19-03) |

## 7. Security Notes

- **Secrets never in committed files.** The `.env` file that gets committed contains public values only.
- **`NEXT_PUBLIC_` is a public channel** — treat naming a var `NEXT_PUBLIC_X` as publishing X.
- **CI builds with a different env than local** are a classic source of "works on my machine" — the five-gate CI above builds with your committed config, so env gaps surface there, not in production.

## 8. Performance Notes

- Turbopack's dev server gives you **Compile/Render split logs** from day one — read them (module 18).
- The production `next build` output is your first performance report: "Collecting page data", "Generating static pages", per-route timings. A build that takes 10 minutes at 50 routes is telling you something.
- `next/font` and `next/image` are configured for you (font in the scaffold, image as a component) — their cost savings compound; modules 15.

## 9. Exercise

**Beginner.** Scaffold `commerce-ops` with the course answers (TS, ESLint, React Compiler, Tailwind, `src/`, App Router, `AGENTS.md`). Then, for **each** prompt answer, write one sentence explaining what it changes in the generated files. Verify by opening the generated `next.config.ts`, `tsconfig.json`, `package.json`, and `eslint.config.mjs`.

**Intermediate.** Add `cacheComponents: true`. Create a page with a `'use cache'` function (`cacheLife('minutes')`) and an uncached `<Suspense>`-wrapped child that reads a fake async delay. Using `next dev` logs and the dev overlay, observe: (a) the cached function running once across two navigations; (b) the overlay's insight about the uncached read if you remove the Suspense wrapper. Screenshot both.

**Production.** Make the CI fail deliberately on each of the five gates in turn (bad lint rule, a type error, a boundary violation, a failing test, a broken build) and record the exact error for each in a `docs/ci-gates.md` file. This is your team's future onboarding doc.

## 10. Architecture Challenge

**Prompt:** A teammate proposes: "Let's skip `src/`, skip React Compiler (too new), keep webpack for 'stability', and enable Biome because it's faster."

Reason in writing:
1. Which of those three rejections have a defensible engineering argument, and what is the actual cost/benefit of each?
2. What is the *team* cost of a project whose build pipeline nobody can explain?
3. If the app is being migrated *from* a webpack Next 14 codebase, does the answer change? (That is the only situation where `--webpack` is legitimate.)

<details>
<summary>Model answer (reason first)</summary>

1. `src/`: cosmetic + convention — the "cost" is zero; rejecting it is a preference, not engineering, and the real cost is mismatch with the course/industry default in a shared codebase. React Compiler: "too new" is no longer true in Sept 2026 — 1.0 has been stable for ~12 months and Next 16 lists support as stable; the honest cost is diagnosing the occasional latent re-render bug it surfaces, which is cheaper than the re-render bugs existing. Webpack: in Next 16 the *default* is Turbopack; opting back out is fighting the framework — "stability" of a bundler you're paying to de-optimize is not a real benefit unless you have a webpack-specific plugin you can't replace (that's the actual test).
2. Every future hire/onboarding/AI agent must reverse-engineer the pipeline; every "why is it slow" starts with "who configured this"; and drift — new framework features (cache handlers, build adapters) are tested against the *default* pipeline, so a custom pipeline accumulates untested deltas. A pipeline you can explain in 30 seconds is worth more than a "faster" one you can't.
3. Yes — during migration, `--webpack` is the supported escape hatch for webpack-specific configs, and the 16 upgrade guide documents exactly that path. The commitment is to remove it, with a date.
</details>

## 11. Official Documentation

- Installation (scaffolder, prompts, system requirements): https://nextjs.org/docs/app/getting-started/installation
- Project structure: https://nextjs.org/docs/app/getting-started/project-structure
- Turbopack: https://nextjs.org/docs/app/api-reference/turbopack
- next.config.ts: https://nextjs.org/docs/app/api-reference/config/next-config-js
- `cacheComponents` config: https://nextjs.org/docs/app/api-reference/config/next-config-js/cacheComponents
- Environment variables: https://nextjs.org/docs/app/guides/environment-variables
- Upgrading to 16 (if migrating): https://nextjs.org/docs/app/guides/upgrading/version-16
- CI build caching: https://nextjs.org/docs/app/guides/ci-build-caching

## 12. What You Should Know Before Continuing

- [ ] I can scaffold and explain every CLI answer in one sentence
- [ ] My scaffold has: TS strict + `noUncheckedIndexedAccess`, ESLint flat config, React Compiler, Tailwind v4, `src/`, `AGENTS.md`, Turbopack
- [ ] `cacheComponents: true` is on and I know what model it enables (to be taught in depth in module 05)
- [ ] My env files are split (`.env` vs `.env.local`) and the secret rules are written down
- [ ] CI runs the five gates in fast→slow order
- [ ] I know `next lint` does not exist in 16 and what replaced it

**Next:** Module 03 — Project Structure: `app/` file conventions, and the `src/` layout that grows with the capstone (each folder justified).
