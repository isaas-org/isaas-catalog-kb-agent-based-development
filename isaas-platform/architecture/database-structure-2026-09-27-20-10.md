# Database structure: one database, one Postgres schema per domain

Scope: platform-wide data layout and the database repository
Status: decided and implemented in `isaas-database` `v1.0.0`

## 1. What was decided

- One Neon database holding three Postgres schemas (namespaces): `vendors`, `items` and `buyer_companies`.
- Cross-domain foreign keys, joins and transactions are allowed and relied on.
- Each schema is the boundary to split along if a domain ever needs its own database.
- All migrations live in one repository, `isaas-org/isaas-database`, separate from every application repo.

## 2. The question that was asked

The three logical parts (item, buyer company, vendor) were originally expected to grow into separate databases. That raised the question of starting with three databases, and possibly three repos, from day one, and whether doing so costs anything.

## 3. Cost of three databases now

1. **Cross-domain FKs and joins are lost, and the schema depends on them.** Examples:
   - The catalog page joins items to vendor status.
   - `reviews` references both `buyer_company_id` and `buyer_user_id`.
   - `subscriptions.pricing_plan_id` points into the item domain.
   - `commission_rates` references both a vendor and an item or category.
2. **Transactions are lost.** A purchase writes `subscriptions` and reads `pricing_plans` in one ACID transaction. Across databases that becomes sagas, compensating writes and eventual consistency.
3. **Three times the operational work:** migrations, connection pools, backups, monitoring and local seeding.
4. **No offsetting benefit yet.** There are no separate teams, no diverging scaling needs and no hard service boundary that would justify the split.

## 4. Options and outcome

| Option | Outcome |
|--------|---------|
| Three separate databases | Rejected, for the costs above |
| One database, everything in `public` | Not chosen. It gives no domain boundary and makes a later split harder |
| One database, one schema per domain | **Chosen.** Logical separation now, full integrity, and a clean extraction boundary later |

Repository outcome: one database means one migrations codebase, so a single repo. It doesn't belong in `isaas-buyer-frontend`, because the database serves buyers, vendors and items alike.

## 5. Catalog visibility is decided at query time

- `items.published_catalog` is a view: `items.status = 'published' AND vendors.status = 'active'`.
- No `is_orderable` flag is cached on items. A cached flag would need a bulk update of every item row whenever a vendor is suspended, with a race window while that runs.
- Backed by a CHECK (a vendor is only `active` while `verified`) and the publish-gate trigger (an item can only be published while its vendor is active).
- Any backend that lists the catalog should read from `items.published_catalog`, not re-implement the filter.

## 6. Ownership rules

- Every domain table is namespaced, for example `items.items`, `vendors.vendors` and `buyer_companies.buyer_users`.
- Tables with cross-domain FKs are created in a dedicated migration after all three core migrations.
- Only the Spring Boot backends connect to the database. The Python agent services and the frontends go through the backend APIs.

## 7. Next steps and triggers to revisit

- Revisit a real split only when one appears: separate team ownership, very different scaling or caching needs per domain, or a hard microservice boundary. The schema to extract would move with `pg_dump --schema`, and any cross-schema FKs would be replaced by application-level checks at that point.
- Decide whether the buyer and vendor backends get separate database roles with grants limited to what each needs, for example read-only access to `vendors` for the buyer backend.
