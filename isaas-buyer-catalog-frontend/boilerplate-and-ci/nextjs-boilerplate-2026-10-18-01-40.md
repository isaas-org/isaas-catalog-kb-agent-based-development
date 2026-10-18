# Next.js boilerplate and CI

Repo: `isaas-org/isaas-buyer-catalog-frontend`
Branch: `feature/FD-5_Initial_boilerplate`
PRs: https://github.com/isaas-org/isaas-buyer-catalog-frontend/pull/1 (merged 2026-10-18 01:40 UTC), https://github.com/isaas-org/isaas-buyer-catalog-frontend/pull/2 (merged 2026-10-18 19:09 UTC)

## 1. What was done

Scaffolded with `create-next-app@16.4.2` (App Router, TypeScript, Tailwind 4, ESLint, `src/` directory), like the vendor frontend. The home page "iSaaS Catalog" shows the buyer backend's health, read through `src/lib/backend.ts` from `BACKEND_API_URL` (default `http://localhost:8081`). The dev server runs on port 3000.

How the backend status works (server-only env, `connection()`, `<Suspense>` under Cache Components) is the same as in the vendor frontend. See `isaas-vendor-catalog-frontend/boilerplate-and-ci/nextjs-boilerplate-2026-10-18-01-42.md`.

It was verified end to end before the PR: with the buyer backend running against Postgres 18, the page rendered `Backend: UP`.

## 2. Aligned with the vendor frontend in a second PR

Only the vendor frontend PR was reviewed, so #1 merged with just the shared logo icons from that review. At the owner's request, PR #2 brought over the rest, on the same `feature/FD-5_Initial_boilerplate` branch after resetting it to `main`:

| Area | After #1 | After #2 (now matches the vendor frontend) |
|------|----------|--------------------------------------------|
| Lockfile | `package-lock.json` committed | Gitignored; `npm install` resolves fresh |
| CI install | `npm ci` with the `setup-node` npm cache | `npm install`, no cache |
| e2e tests | None | Playwright smoke test and `e2e` CI job |
| `AGENTS.md` docs pointer | Next.js-managed block pointing at `node_modules/next/dist/docs/` | Hand-written section pointing at https://nextjs.org/docs, with `agentRules: false` |

The only intended differences from the vendor frontend are names, ports (3000, with the backend on 8081) and page text:

```ts
// e2e/home.spec.ts
await expect(page).toHaveTitle("iSaaS Catalog");
await expect(page.getByRole("heading", { level: 1 })).toHaveText("iSaaS Catalog");
```

Verified before the PR: a fresh `npm install` with the lockfile staying ignored, lint, and `npm run test:e2e` against a production build. CI passed on both jobs.

The README documents the `npm audit` finding from `braces` (lint tooling only) and the decision to keep it until Next.js unpins `fast-glob`.

## 3. Decisions

| Decision | Chosen | Why |
|----------|--------|-----|
| Data access | Only through the buyer backend API | One place for business rules; no database credentials in a Vercel app |
| Runtime config | `BACKEND_API_URL` read at request time | One build promoted across environments |
| Icons | Shared iSaaS logo | Same brand mark on both apps |
| Review outcomes | Mirrored from the vendor frontend | Both frontends should build, test and document themselves the same way |

## 4. Next steps

- Catalog pages: browse by category, search, filters (integrations, certifications, industries), item detail, and comparison. Public pages use SSR or ISR for SEO.
- e2e tests for each page as it is added.
