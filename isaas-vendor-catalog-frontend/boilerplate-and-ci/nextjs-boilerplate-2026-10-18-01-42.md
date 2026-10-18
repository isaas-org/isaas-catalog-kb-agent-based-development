# Next.js boilerplate, e2e tests and CI

Repo: `isaas-org/isaas-vendor-catalog-frontend`
Branch: `feature/FD-5_Initial_boilerplate`
PR: https://github.com/isaas-org/isaas-vendor-catalog-frontend/pull/1 (merged 2026-10-18 01:42 UTC)

## 1. What was done

Scaffolded with `create-next-app@16.4.2`: App Router, TypeScript, Tailwind 4, ESLint and a `src/` directory. The template page was replaced with an "iSaaS Vendor Portal" page that shows the vendor backend's health. The PR review then added Playwright e2e tests, a no-lockfile dependency policy, the iSaaS logo icons, and a hand-maintained `AGENTS.md`.

```
src/app/            layout.tsx, page.tsx, globals.css, favicon.ico, icon.png, apple-icon.png
src/components/     BackendStatus.tsx
src/lib/backend.ts  server-side client for isaas-vendor-catalog-mgmt-backend
e2e/home.spec.ts    Playwright smoke test
playwright.config.ts, next.config.ts, .github/workflows/test.yml
```

## 2. Talking to the backend

`BACKEND_API_URL` has no `NEXT_PUBLIC_` prefix, so it's never inlined into the browser bundle. It's read at request time, so one build works across environments:

```tsx
// src/components/BackendStatus.tsx
export async function BackendStatus() {
  await connection();          // opt into request time before reading env
  let status: string;
  try {
    status = (await getBackendHealth()).status;
  } catch (err) {
    status = err instanceof Error ? err.message : "unreachable";
  }
  ...
}
```

`page.tsx` wraps it in `<Suspense>`, which Cache Components (`cacheComponents: true`) requires for uncached data. The build output marks `/` as a partial prerender: a static shell, with the status streamed in at request time. These APIs are new in Next.js 16, so the bundled docs in `node_modules/next/dist/docs/` were read before writing this code.

## 3. Review changes

| Comment | Outcome |
|---------|---------|
| Groundwork for integration tests | Playwright with one smoke test against a production build, plus a separate `e2e` CI job that uploads the HTML report when a run fails |
| Point `AGENTS.md` at online docs, not `node_modules` | Done. Needed `agentRules: false` in `next.config.ts`, because `next dev` rewrites anything between its `nextjs-agent-rules` markers. Checked that `next dev` now leaves the file unchanged |
| Gitignore `package-lock.json` | Done |
| `npm install` instead of `npm ci` in CI | Done, and the `setup-node` npm cache was removed |
| Favicon from the logo | The logo's checkerboard was baked into the RGB pixels. It was keyed out by saturation (background at or below 3, logo at or above 50) to make transparent icons: `favicon.ico` (16/32/48), `icon.png` (512), `apple-icon.png` (180 on white) |
| Purple gradient for headings and buttons | Deferred by the owner. The thread stays open as a reminder |

## 4. e2e setup

```ts
// playwright.config.ts
webServer: {
  command: "npm run build && npm run start",
  url: `http://localhost:${port}`,
  env: { BACKEND_API_URL: process.env.BACKEND_API_URL ?? "http://localhost:59999" },
},
```

By default the tests point the app at a port nothing listens on, so they don't need a running backend. The smoke test checks that the title, the heading and the streamed backend status render. Pointing `BACKEND_API_URL` at a real backend makes the same tests run against it.

## 5. Decisions

| Decision | Chosen | Why |
|----------|--------|-----|
| Lockfile | None; `npm install` every run | Owner's preference for always-fresh dependencies. Framework packages stay pinned exactly |
| `npm audit` findings | Accepted for now | Dev-only lint tooling; no patched `braces` exists; production dependencies are clean |
| Next.js agent rules block | Disabled, with our own docs section | The managed block points at `node_modules`, which a fresh clone doesn't have |
| e2e browser | Chromium only | Enough for groundwork; more browsers can be added as projects |

## 6. Next steps

- Vendor pages: listing editor (draft item, pricing plans, features, media), then publish.
- Auth0 login, once the backend supports it.
- Brand theme tokens for the purple gradient (the open review thread).
- e2e tests against a mocked or real vendor backend as pages are added.
