# isaas-catalog-kb-agent-based-development

Knowledge base for iSaaS, an open-source "Amazon for SaaS" marketplace. Buyer companies discover and buy SaaS products, vendors list and sell them, and the platform runs the marketplace in between.

The code lives in separate repos. This repo records, per repo and per feature, what was built, the decisions behind it (including the options that were rejected), and what comes next. It's written so that a developer or a coding agent can pick up any part of the project without the original conversations.

## Layout

```
<repo-name>/<feature-folder>/<doc-name>-YYYY-MM-DD-HH-MM.md   work done in one repo
isaas-platform/language-and-framework/...                     language, framework and provider choices
isaas-platform/architecture/...                               cross-repo architecture
```

The timestamp in each file name is when the documented work or decision happened, in UTC, so docs in a folder sort chronologically.

## Contents

### `isaas-database/schema-and-seed-data`

| Doc | Covers |
|-----|--------|
| `catalog-schema-design-2026-09-27-03-07.md` | Item, buyer company and vendor tables, and why each was shaped that way |
| `migrations-and-release-pipeline-2026-09-28-00-07.md` | dbmate migrations, database-enforced rules, tests, tag-based release to Neon (`v1.0.0`) |
| `catalog-seed-data-injection-2026-10-11-03-17.md` | Sample vendors, items and taxonomy, and the manual seed workflow |
| `auth-membership-schema-2026-10-18-20-34.md` | FD-8: staff, Auth0 identity columns, buyer verification gate, join requests, access tokens, test-user seeds |

### `isaas-platform/language-and-framework`

| Doc | Covers |
|-----|--------|
| `database-engine-and-provider-2026-09-27-19-59.md` | PostgreSQL over MySQL and Oracle, Neon over RDS, Supabase and others |
| `application-stack-2026-09-27-20-46.md` | Spring Boot backends, Next.js frontends on Vercel, Python MCP agent services |
| `auth-provider-2026-09-27-20-49.md` | Auth0 as an identity-only provider |
| `app-repos-versions-and-ci-2026-10-17-22-17.md` | The six app repos: versions, ports, shared CI conventions, CI secrets and frontend dependency policy |

### `isaas-platform/architecture`

| Doc | Covers |
|-----|--------|
| `database-structure-2026-09-27-20-10.md` | One database with a Postgres schema per domain, and catalog visibility decided at query time |
| `service-boundaries-2026-09-27-20-46.md` | How frontends, backends, agent services and the database talk to each other |
| `auth-and-membership-2026-09-27-20-55.md` | How people join a buyer company or vendor: bootstrap, peer approval, SSO (first design) |
| `authn-authz-design-2026-10-18-20-20.md` | Auth0 setup, membership only from the database, verification of company and person, buyer verification as a hard gate, staff |
| `agent-auth-and-test-users-2026-10-18-20-20.md` | Personal access tokens for agent apps now, OAuth later; durable Auth0 test users and the test-user matrix |

### Application repos (`<repo>/boilerplate-and-ci`)

| Doc | Covers |
|-----|--------|
| `isaas-buyer-catalog-browsing-backend/.../spring-boot-boilerplate-2026-10-18-02-11.md` | Spring Boot setup, local vs deployed configuration, CI with secrets |
| `isaas-vendor-catalog-mgmt-backend/.../spring-boot-boilerplate-2026-10-18-02-10.md` | Review outcomes: CI secrets with a `check-config` guard, local-config note, publish-gate rule |
| `isaas-buyer-catalog-frontend/.../nextjs-boilerplate-2026-10-18-01-40.md` | Next.js setup, and the follow-up PR that aligned it with the vendor frontend |
| `isaas-vendor-catalog-frontend/.../nextjs-boilerplate-2026-10-18-01-42.md` | Request-time backend status, Playwright e2e, no-lockfile policy, logo icons, `AGENTS.md` docs pointer |
| `isaas-buyer-catalog-browsing-agent-app/.../mcp-server-boilerplate-2026-10-18-01-49.md` | Buyer MCP server settings and the end-to-end check |
| `isaas-vendor-catalog-mgmt-agent-app/.../mcp-server-boilerplate-2026-10-18-01-48.md` | MCP SDK 2.x, tools as classes under `tools/`, no nested functions, testing approach |

## Current state

- The database schema is live on Neon as `v1.0.0`, with sample catalog data seeded.
- All six application repos have merged boilerplate with green CI: each app starts, reaches its backend or database, and reports health. No catalog features yet.
- The auth and membership schema (FD-8) is built and in review as `isaas-database` PR #3. Backends (FD-7), frontends (FD-6) and agent apps (FD-11) follow.

## Contributing

Conventions for writing docs and working on this repo are in [`AGENTS.md`](AGENTS.md): folder and file naming, size limits per doc, code snippets alongside explanations, and feature-branch-only work. `CLAUDE.md` imports the same file for Claude Code sessions that can't read `AGENTS.md` directly.
