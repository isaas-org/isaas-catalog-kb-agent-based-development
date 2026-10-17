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

### `isaas-platform/language-and-framework`

| Doc | Covers |
|-----|--------|
| `database-engine-and-provider-2026-09-27-19-59.md` | PostgreSQL over MySQL and Oracle, Neon over RDS, Supabase and others |
| `application-stack-2026-09-27-20-46.md` | Spring Boot backends, Next.js frontends on Vercel, Python MCP agent services |
| `auth-provider-2026-09-27-20-49.md` | Auth0 as an identity-only provider |

### `isaas-platform/architecture`

| Doc | Covers |
|-----|--------|
| `database-structure-2026-09-27-20-10.md` | One database with a Postgres schema per domain, and catalog visibility decided at query time |
| `service-boundaries-2026-09-27-20-46.md` | How frontends, backends, agent services and the database talk to each other |
| `auth-and-membership-2026-09-27-20-55.md` | How people join a buyer company or vendor: bootstrap, peer approval, SSO (designed, not yet migrated) |

## Current state

- The database schema is live on Neon as `v1.0.0`, with sample catalog data seeded.
- The application repos (buyer and vendor frontends, backends and agent services) are created but empty. Boilerplate is the next step.
- The auth and membership schema additions are designed but not yet migrated.

## Contributing

Conventions for writing docs and working on this repo are in [`AGENTS.md`](AGENTS.md): folder and file naming, size limits per doc, code snippets alongside explanations, and feature-branch-only work. `CLAUDE.md` imports the same file for Claude Code sessions that can't read `AGENTS.md` directly.
