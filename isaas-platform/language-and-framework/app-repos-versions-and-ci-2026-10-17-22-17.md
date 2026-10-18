# Application repos: versions, ports and CI conventions

Scope: the six application repos created for the catalog feature
Status: boilerplate merged to `main` in all six repos on 2026-10-18 (branch `feature/FD-5_Initial_boilerplate`)

## 1. What was done

The stack chosen in `application-stack-2026-09-27-20-46.md` was turned into six repos, one per side and layer. Each `main` started with a one-line README. The boilerplate went in through a PR from `feature/FD-5_Initial_boilerplate`, was reviewed by the repo owner, and was merged.

| Repo | Role | Stack | Local port |
|------|------|-------|-----------|
| `isaas-buyer-catalog-browsing-backend` | Buyer catalog API | Java 21, Spring Boot 4.1.1, Maven | 8081 |
| `isaas-vendor-catalog-mgmt-backend` | Vendor catalog API | Java 21, Spring Boot 4.1.1, Maven | 8082 |
| `isaas-buyer-catalog-frontend` | Buyer web app | Next.js 16.4.2, React 19.3, TypeScript, Tailwind 4 | 3000 |
| `isaas-vendor-catalog-frontend` | Vendor web app | Next.js 16.4.2, React 19.3, TypeScript, Tailwind 4 | 3001 |
| `isaas-buyer-catalog-browsing-agent-app` | Buyer MCP server | Python 3.13, uv, MCP SDK 2.3, httpx | 8091 |
| `isaas-vendor-catalog-mgmt-agent-app` | Vendor MCP server | Python 3.13, uv, MCP SDK 2.3, httpx | 8092 |

The ports don't overlap, so the full buyer and vendor stacks can run side by side on one machine against `isaas-database`'s docker Postgres on 55432.

## 2. Version choices

| Choice | Picked | Why |
|--------|--------|-----|
| Java | 21 (LTS) | The JDK available where the boilerplate was built and verified. Java 25 is also offered by Spring Initializr and is a possible later upgrade |
| Spring Boot | 4.1.1 | Current release on Spring Initializr. Initializr wrote it as `4.1.1.RELEASE`, which doesn't exist on Maven Central; the POM uses `4.1.1` |
| Next.js | 16.4.2 | Latest `create-next-app`. Ships Cache Components and partial prerendering on by default |
| MCP Python SDK | `mcp>=2.3,<3` | Current major. In 2.x `FastMCP` is renamed `MCPServer`, so v1 examples don't apply |
| Python | 3.13 | Current stable, managed with uv and a committed `uv.lock` |

## 3. Shared conventions in every repo

- `AGENTS.md` holds the repo's rules for coding agents, and `CLAUDE.md` is one line, `@AGENTS.md`, for Claude Code sessions that can't read `AGENTS.md` directly.
- `README.md` covers local run, configuration and checks.
- CI is `.github/workflows/test.yml`, running on every PR and on every push to `feature/**`, the same trigger as `isaas-database`.
- Never push to `main`; work on the feature branch the repo owner names.
- Only the backends talk to the database. Frontends and agent apps reach data through their side's backend API.

## 4. CI database credentials come from organization secrets

The backend CI starts a throwaway `postgres:18` service container. Its credentials were first hardcoded in the workflow. In review, the owner chose to move them into GitHub secrets, which are now set once as **organization secrets** on `isaas-org`:

```yaml
services:
  postgres:
    image: postgres:18
    env:
      POSTGRES_USER: ${{ secrets.CI_DATABASE_USERNAME }}
      POSTGRES_PASSWORD: ${{ secrets.CI_DATABASE_PASSWORD }}
      POSTGRES_DB: isaas_test
env:
  DATABASE_URL: ${{ secrets.CI_DATABASE_URL }}
```

Trade-offs that were discussed:
- A missing or misnamed secret would stop Postgres from starting, with an unclear error. A `check-config` job now runs first and fails with a message naming the missing secret. It also fails if `CI_DATABASE_URL` doesn't point at `isaas_test`.
- GitHub doesn't pass secrets to workflows triggered from forks. Fork PRs fail `check-config`, and a maintainer has to push the change to a branch in the repo to test it.

## 5. Frontend dependency policy

Decided in review of the vendor frontend:

- No `package-lock.json` in the repo; it is gitignored.
- CI runs `npm install` without the `setup-node` npm cache, so every run resolves dependencies fresh.
- `next`, `react` and `react-dom` stay pinned to exact versions in `package.json`, so the framework doesn't move between runs.
- Accepted risk: a new release of a transitive dependency can land in any build.

`npm audit` reports 5 high-severity findings from `braces` (GHSA-vfj7-8cjw-p6xm), reached through `eslint-config-next` -> `@next/eslint-plugin-next` -> `fast-glob@3.3.1`, which Next.js pins. No patched `braces` exists, and the only offered fix downgrades `eslint-config-next` to 14.x. Kept as is: it is lint tooling only, and `npm audit --omit=dev` is clean. Recheck on each Next.js upgrade.

## 6. Agent app conventions

Decided in review of the vendor agent app:

- Related MCP tools live together in one module under `tools/`, as a class that takes its dependencies in `__init__` and registers its methods in `register()`. `server.py` only wires these classes together.
- No nested functions in source or tests.
- Each tool module has a matching test module under `tests/tools/`.

See the agent app docs in `isaas-*-agent-app/boilerplate-and-ci/`.

## 7. Next steps

- Bring `isaas-buyer-catalog-frontend` in line with the vendor frontend: Playwright e2e, no lockfile, `npm install` in CI, and online Next.js docs in `AGENTS.md`. It currently only has the shared logo icons.
- Add Spring Security with Auth0 to both backends, after the auth and membership schema is migrated in `isaas-database`.
- Add brand colors (a purple gradient from the logo) to the frontends; deferred in review.
- Decide hosting for the backends and agent apps; the frontends go to Vercel.
