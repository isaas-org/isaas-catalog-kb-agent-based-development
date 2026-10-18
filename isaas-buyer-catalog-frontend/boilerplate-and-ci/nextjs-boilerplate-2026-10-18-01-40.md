# Next.js boilerplate and CI

Repo: `isaas-org/isaas-buyer-catalog-frontend`
Branch: `feature/FD-5_Initial_boilerplate`
PR: https://github.com/isaas-org/isaas-buyer-catalog-frontend/pull/1 (merged 2026-10-18 01:40 UTC)

## 1. What was done

Scaffolded with `create-next-app@16.4.2` (App Router, TypeScript, Tailwind 4, ESLint, `src/` directory), like the vendor frontend. The home page "iSaaS Catalog" shows the buyer backend's health, read through `src/lib/backend.ts` from `BACKEND_API_URL` (default `http://localhost:8081`). The dev server runs on port 3000.

How the backend status works (server-only env, `connection()`, `<Suspense>` under Cache Components) is the same as in the vendor frontend. See `isaas-vendor-catalog-frontend/boilerplate-and-ci/nextjs-boilerplate-2026-10-18-01-42.md`.

It was verified end to end before the PR: with the buyer backend running against Postgres 18, the page rendered `Backend: UP`.

## 2. State at merge

Only the vendor frontend PR was reviewed. Of its review changes, only the shared logo icons were applied here. So this repo, as merged, still has:

| Area | Buyer frontend (as merged) | Vendor frontend |
|------|-----------------------------|-----------------|
| Lockfile | `package-lock.json` committed | Gitignored |
| CI install | `npm ci` with the `setup-node` npm cache | `npm install`, no cache |
| e2e tests | None | Playwright smoke test and `e2e` CI job |
| `AGENTS.md` docs pointer | Next.js-managed block pointing at `node_modules/next/dist/docs/` | Hand-written section pointing at https://nextjs.org/docs, with `agentRules: false` |
| Icons | Logo icons | Logo icons |

The README documents the `npm audit` finding from `braces` (lint tooling only) and the decision to keep it until Next.js unpins `fast-glob`.

## 3. Decisions

| Decision | Chosen | Why |
|----------|--------|-----|
| Data access | Only through the buyer backend API | One place for business rules; no database credentials in a Vercel app |
| Runtime config | `BACKEND_API_URL` read at request time | One build promoted across environments |
| Icons | Shared iSaaS logo | Same brand mark on both apps |

## 4. Next steps

- Bring this repo in line with the vendor frontend's review outcomes (table above).
- Catalog pages: browse by category, search, filters (integrations, certifications, industries), item detail, and comparison. Public pages use SSR or ISR for SEO.
- e2e tests for each page as it is added.
