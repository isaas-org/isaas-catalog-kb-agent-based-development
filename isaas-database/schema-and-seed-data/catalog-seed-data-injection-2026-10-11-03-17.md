# Catalog seed data injection

Repo: `isaas-org/isaas-database`
Branch: `feature/FD-4_Catalog_data_injection`
PR: https://github.com/isaas-org/isaas-database/pull/2 (merged)
Applied to Neon: `seed` workflow run 38108955989, 2026-10-11 03:38 UTC, conclusion `success`

## 1. Why

The schema (`v1.0.0`) was live on Neon but empty. The first user-facing goal is to view and use the catalog as a buyer, which needs real-looking data to build and test the buyer backend and frontend against.

## 2. What was done

```
db/seeds/
  01_taxonomy.sql       categories (5 + 1 subcategory), industries, integrations, certifications
  02_vendors.sql        5 vendors, one admin vendor_user each
  03_items.sql          6 items across the vendors
  04_item_details.sql   pricing plans, features, taxonomy links, media
scripts/seed.sh         applies db/seeds/[0-9]*.sql in order, prints the published_catalog row count
.github/workflows/seed.yml   manual (workflow_dispatch) run against Neon
README.md               new "Seed data" section
```

## 3. The data

| Vendor | Status | Items |
|--------|--------|-------|
| Northstar CRM, Inc. | active / verified | Northstar CRM (published) |
| Pulseboard, Inc. | active / verified | Pulseboard Analytics (published) |
| Everhire, Inc. | active / verified | Everhire ATS (published), Everhire Payroll (draft) |
| Northwind DevOps Co. | active / verified | Northwind Pipelines (published) |
| Cascade Labs | pending / pending | Cascade Workflow (draft) |

Categories: CRM, Project Management, Data Analytics (with subcategory Business Intelligence), HR & People, DevOps & Infrastructure.

Expected result: 4 rows in `items.published_catalog`. There are no reviews yet, so `avg_rating` is NULL and `review_count` is 0 on every item. All URLs and emails use the `.test` domain, and logos are `placehold.co` placeholders.

## 4. Decisions

| Decision | Chosen | Why |
|----------|--------|-----|
| Seed as migrations or separately | Separately, in `db/seeds/` | It's sample data, not schema. It shouldn't create a version tag or be tracked by dbmate |
| How seeds reach Neon | Manual GitHub Action (`workflow_dispatch`) | Reuses the `production` environment's `NEON_DATABASE_URL` secret and required-reviewer gate, so the connection string is never shared with Claude |
| Re-runnable | Every insert uses `ON CONFLICT` on the natural key (slug); media uses `NOT EXISTS` | Running the seed twice leaves the same rows |
| Vendor states | Seeded directly as active/verified, bypassing the real KYB flow | The publish gate only allows published items from active, verified vendors |
| One pending vendor and two drafts | Kept on purpose | Proves the publish gate and the catalog view's filtering on real data |

## 5. Verification

Before the PR was opened, everything was run locally with Docker (`postgres:18`):

- all migrations applied cleanly;
- `seed.sh` produced 4 rows in `items.published_catalog`, with the draft item and the pending vendor's item excluded;
- a second run gave identical row counts;
- the full `scripts/test.sh` suite still passed.

After merging, the user started the `seed` workflow and approved the `production` gate. The run finished successfully, with "Apply seed data" running from 03:38:05 to 03:38:09 UTC. The row count wasn't queried directly on Neon afterwards.

## 6. Running it again

```bash
DATABASE_URL=... ./scripts/seed.sh    # local docker database or a Neon branch
```

For production Neon, run the **seed** workflow from the Actions tab and approve the job. It creates no tag.

## 7. Next steps

- Optionally confirm `SELECT count(*) FROM items.published_catalog` returns 4 on Neon.
- Build the buyer catalog backend and frontend against `items.published_catalog`.
- Add sample reviews and subscriptions later, once the buyer side can create them, or seed them to test ratings and verified-purchase logic.
