# Database engine and provider: PostgreSQL on Neon

Scope: platform-wide (used by `isaas-database` and every backend that reads it)
Status: decided and in use. Neon project created, schema `v1.0.0` deployed.

## 1. What was decided

- **Engine:** PostgreSQL (version 18 on Neon, `postgres:18` locally)
- **Provider:** Neon (serverless Postgres)

The user had already leaned towards a relational database and accepted the cost of adding columns and backfilling rows as the schema evolves.

## 2. Engine options compared

| Option | Verdict | Reasoning |
|--------|---------|-----------|
| Oracle | Rejected | Per-core licensing and support contracts, built for large enterprises with DBA teams. Nothing in the schema needs Oracle-specific features |
| MySQL | Reasonable, but weaker for this schema | JSON support is less mature for indexed querying, full-text search is more limited, and native ENUM/CHECK support has historically been weaker |
| PostgreSQL | Chosen | See below |

Why PostgreSQL fits:

- Native `JSONB` with GIN indexes, used for `pricing_plans.usage_limits` and available for future flexible fields.
- Built-in full-text search (`tsvector`/`tsquery`) and trigram matching (`pg_trgm`) cover the "search and filtering" and "comparison shopping" use cases without a separate search service on day one.
- Native `ENUM` types and mature `CHECK` constraints suit the many status and role fields.
- Strong relational integrity for a schema with many FKs, junction tables and nullable-FK override patterns (`commission_rates`).
- Free, open source and a first-class target for every ORM.

## 3. Provider options compared

| Option | Verdict | Reasoning |
|--------|---------|-----------|
| AWS RDS for PostgreSQL | Strong alternative | Mature and the industry standard, but bills for a running instance 24/7 and has no database branching |
| AWS Aurora PostgreSQL | Rejected for now | Proprietary storage engine, higher price, overkill before real production load |
| Google Cloud SQL / Azure Database for PostgreSQL | Rejected | Only worth it if already committed to that cloud |
| Supabase | Rejected | Bundles auth, storage and auto-generated APIs. The user chose to keep the database separate from those concerns |
| Neon | Chosen | See below |

## 4. Inputs that decided the provider

- **Cloud commitment:** none (greenfield).
- **Bundling:** keep the database separate from auth and file storage, which ruled out Supabase.
- **Neon over RDS** because:
  - instant copy-on-write branches fit the iterative "add a column, backfill, test" workflow;
  - scale-to-zero pricing suits a pre-revenue project;
  - it's stock Postgres, so moving to RDS or Aurora later is a standard `pg_dump`/`pg_restore` or logical replication job, not a rewrite.
- **Risk accepted:** Neon is a younger company than AWS.

## 5. How Neon is used

- One Neon project with the default branch `main` and the default database. Neon's branches are separate from git branches.
- The `vendors`, `items` and `buyer_companies` schemas are created by migrations, not in the Neon console.
- **Direct** (non-pooled) connection string for migrations, stored only as the `NEON_DATABASE_URL` secret on the `production` environment of `isaas-database`. It was never pasted into chat.
- The **pooled** connection string is meant for application runtime.

## 6. Implications for backend code

- Spring Boot with Hibernate doesn't map native Postgres ENUMs cleanly out of the box. Use the `hibernate-types` library or `@Convert` mappings (see `application-stack-2026-09-27-20-46.md`).
- Use the pooled Neon host for application connections, and the direct host only for migrations.

## 7. Next steps

- Consider the Neon GitHub integration for a throwaway database branch per PR.
- Revisit RDS or Aurora only once there is real production load or a compliance need (live payouts, VPC peering).
