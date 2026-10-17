# Migrations, database rules and the release pipeline

Repo: `isaas-org/isaas-database`
Branch: `feature/FD-2` (built in a local Claude Code session, merged by the repo owner)
Release: `v1.0.0`, deployed to Neon by the `deploy` workflow on 2026-09-28 00:07 UTC

## 1. What was done

The schema from `catalog-schema-design-2026-09-27-03-07.md` was implemented as plain SQL migrations, together with a test suite and a release pipeline that applies the migrations to Neon.

```
db/migrations/
  20260927100000_create_schemas_and_extensions.sql   schemas, pg_trgm, set_updated_at()
  20260927100100_create_vendors_core.sql
  20260927100200_create_items_core.sql                (depends on vendors)
  20260927100300_create_buyer_companies_core.sql      (depends on items.industries)
  20260927100400_create_cross_domain_tables.sql       subscriptions, watchlist, reviews, commission_rates
  20260927100500_create_catalog_search_and_view.sql   full-text search, trigram index, published_catalog
```

Tables with cross-domain foreign keys are created last, once all three domains' core tables exist.

## 2. Tooling choices

| Choice | Picked | Why |
|--------|--------|-----|
| Migration tool | dbmate (`v2.36.0` in CI) | Plain SQL with `-- migrate:up` / `-- migrate:down` sections, and it doesn't depend on whichever backend framework is chosen |
| Primary keys | `uuid DEFAULT uuidv7()` | Time-ordered, so indexes stay compact, and IDs stay unique if a domain is ever split into its own database |
| Local database | `docker-compose.yml` with `postgres:18` on host port 55432 | Matches Neon's major version and avoids clashing with a local Postgres on 5432 |
| `updated_at` | One shared trigger function, `public.set_updated_at()` | Same behavior on every mutable table |

## 3. Rules enforced by the database

| Rule | How |
|------|-----|
| A vendor can only be `active` while `verified` | CHECK `vendors_active_requires_verified` |
| An item can only become `published` while its vendor is `active` | trigger `items.enforce_publish_gate` (also stamps `published_at`) |
| The catalog shows only published items from active vendors | view `items.published_catalog`, a join decided at query time |
| `reviews.is_verified_purchase` can't be forged | trigger sets it at insert (paid subscription only; trials don't count) and freezes it afterwards |
| `avg_rating` / `review_count` stay exact | trigger on `reviews` recomputes them under a row lock |
| The category tree has no cycles | trigger `items.prevent_category_cycle` |
| A review author or artifact submitter belongs to the company named on the row | composite FKs to `(id, buyer_company_id)` / `(id, vendor_id)` |
| A subscription's plan belongs to its item | composite FK to `pricing_plans (id, item_id)` |
| A commission rate has one scope, and each scope has one rate | CHECK `num_nonnulls(...) <= 1` + `UNIQUE NULLS NOT DISTINCT` |
| At most one default payment method / payout account | partial unique indexes `WHERE is_default` |
| A subscribed plan or a vendor with items can't be hard-deleted | `ON DELETE RESTRICT`; retire with status fields instead |

Commission resolution goes from most to least specific: item, then category, then vendor-wide, then the platform default from config.

Search: `items.search_vector` is a generated `tsvector`, weighted name > tagline > description, with a GIN index. A `pg_trgm` GIN index on `name` gives typo-tolerant matching.

## 4. Tests

`scripts/test.sh` runs against a throwaway database (`TEST_DATABASE_URL`). It:

1. drops and recreates the database, then applies every migration;
2. runs each `tests/NN_*.sql` file inside a transaction that is rolled back afterwards, using shared fixtures from `tests/_fixtures.sql`;
3. rolls back every migration and checks nothing is left behind;
4. migrates up again and re-runs the tests.

The 11 test files cover catalog visibility, vendor gates, reviews, commission rates, subscriptions, verification artifacts, payments and payouts, items and taxonomy, users and watchlist, search, and enum vocabulary. Conventions:

- Fixture rows are referenced by name, for example `pg_temp.fx('vendor.pending')`.
- A negative test must name the constraint or trigger that should reject the statement (`pg_temp.assert_raises`), so it can't pass because a different constraint happened to fire.
- `tests/11_enum_vocabulary.sql` pins every enum value list. Changing an enum means updating that file.

CI (`.github/workflows/test.yml`) runs the same script against a `postgres:18` service container on every PR and every push to `feature/**`.

## 5. Release flow (`deploy.yml`)

This implements the requirement that schemas are versioned with tags as they reach `main`, and that a pipeline, not a person, applies them to Neon.

1. Work happens on a feature branch with a PR. CI runs the tests.
2. A PR that touches `db/migrations/` needs exactly one label, `release:major`, `release:minor` or `release:patch`. `release-label.yml` fails the PR until it has one.
3. Merging to `main` triggers `deploy.yml`, which only ever deploys from `main`:
   - **test**: re-runs the suite.
   - **plan**: reads the merged PR's label (defaulting to minor), computes the next version (first release `v1.0.0`) and writes a release plan to the run summary.
   - **migrate**: waits for approval from the `production` environment's required reviewers. It then snapshots the applied migrations, creates the tag, runs `dbmate up` and verifies nothing is pending.
4. If applying or verifying fails, migrate rolls back to the snapshot and deletes the tag it created.
5. A run with no pending migrations creates no tag. A manual `workflow_dispatch` run (main only) can override the bump.

Release logic is in `scripts/deploy.sh`. `scripts/test_deploy.sh` exercises the failure paths in CI.

Version meaning: major for breaking changes (dropping or renaming a column, table or enum value), minor for additive changes (new table, nullable column, enum value), patch for no structural change (index, comment, trigger fix).

## 6. Working rules and setup

- Claude never pushes to `main`. It only pushes to feature branches the user names. Merging and promotion are done by the user.
- Branch naming: `feature/FD-<n>` or `feature/FD-<n>_<Description>`.
- Migrations are append-only once released. Never edit one that has been applied to Neon; add a new one.
- A `-- migrate:down` section must genuinely undo its `up` section, because rollback depends on it. For destructive changes, take a Neon branch snapshot first.
- One-time setup (done): the `production` environment holds the `NEON_DATABASE_URL` secret (Neon's direct, non-pooled connection string with `sslmode=require`), and the user is its required reviewer.

## 7. Next steps

- A new migration for the Part 2 auth and membership schema (`release:minor`).
- The deferred tables listed in `catalog-schema-design-2026-09-27-03-07.md`.
- Optional: install Neon's GitHub integration for a throwaway Neon branch per PR.
