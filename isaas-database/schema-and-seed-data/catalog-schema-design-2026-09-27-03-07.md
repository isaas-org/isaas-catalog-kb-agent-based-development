# Catalog schema design (item, buyer company, vendor)

Repo: `isaas-org/isaas-database`
Feature: initial database schema for the catalog
Status: designed, implemented and released as `v1.0.0` (see `migrations-and-release-pipeline-2026-09-28-00-07.md`)

## 1. Context

iSaaS is an "Amazon for SaaS" marketplace. Buyer companies (BCs) discover and buy SaaS products, vendors list and sell them, and the platform is the operator in between. The business vocabulary came from the business use-cases and glossary PDF (`saas_marketplace_business_terms.pdf`), produced before any design work started.

The first feature chosen for development was the catalog (data, backend, frontend), starting from the data. The data was split into three logical parts:

- **item**: the SaaS product or listing
- **buyer company**: the organization buying software, plus its people
- **vendor**: the organization selling software, plus its people

Each part started as a core table with surrounding tables for entities that naturally split off it. The plan was to let each part grow into its own database later if needed. That was later revised to one database with one Postgres schema per domain (see `isaas-platform/architecture/database-structure-2026-09-27-20-10.md`).

## 2. Domains as implemented

| Postgres schema | Tables |
|-----------------|--------|
| `items` | `items`, `categories`, `item_categories`, `industries`, `pricing_plans`, `features`, `integrations`, `item_integrations`, `certifications`, `item_certifications`, `item_media`, `reviews`, view `published_catalog` |
| `buyer_companies` | `buyer_companies`, `buyer_users`, `addresses`, `payment_methods`, `buyer_verification_artifacts`, `subscriptions`, `watchlist` |
| `vendors` | `vendors`, `vendor_users`, `vendor_addresses`, `vendor_verification_artifacts`, `payout_accounts`, `commission_rates` |

Naming differences from the design discussion:

- The buyer schema is `buyer_companies`, not `buyers` as written in the Part 1 plan. Tables are referenced as `buyer_companies.buyer_users`, `buyer_companies.subscriptions`, and so on.
- `company_size` has six bands (`1-10`, `11-50`, `51-200`, `201-1000`, `1001-5000`, `5000+`) instead of the four discussed (`1-10`, `11-50`, `51-200`, `200+`).
- `reviews` lives in the `items` schema.

## 3. Item domain

**`items.items`**: `id, vendor_id, name, slug, tagline, description, logo_url, website_url, status (draft/published/suspended/archived), published_at, avg_rating, review_count, search_vector, created_at, updated_at`

Surrounding tables:

- `categories`: self-referencing `parent_category_id` for subcategories. `item_categories` makes it many-to-many.
- `pricing_plans`: `tier_name, pricing_model (flat/seat/usage/freemium), price, currency (default USD), billing_frequency (monthly/annual, NULL only for freemium), usage_limits (jsonb), is_active`. One plan is one SKU in glossary terms.
- `features`: one row per item and feature, with `feature_name`, `feature_value` and `description`.
- `integrations` + `item_integrations`, `certifications` + `item_certifications`: shared lookups with junction tables.
- `industries`: lookup referenced by `buyer_companies.industry_id`. The design referenced it but never defined it, so it was added during implementation.
- `item_media`: `media_type (image/video/document), url, sort_order`.
- `reviews`: `item_id, buyer_company_id, buyer_user_id, rating, body, is_verified_purchase`.

## 4. Buyer company domain

- `buyer_companies`: `name, website_url, industry_id, company_size, status (active/suspended), verification_status (unverified/pending/verified/rejected)`
- `buyer_users`: `buyer_company_id, name, email, role (admin/purchaser/approver/viewer)`. Email is unique case-insensitively.
- `addresses`: `type (billing/headquarters)` plus postal fields.
- `payment_methods`: `type (card/ach/purchase_order), provider_token, is_default`. Only processor tokens are stored, never raw card data. Purchase orders have no token.
- `buyer_verification_artifacts`: `artifact_type (business_registration/tax_id/proof_of_address/other), file_url, status (pending/approved/rejected), submitted_by_user_id, reviewed_by_staff_id (nullable, no FK yet), reviewed_at, review_notes`.
- `subscriptions`: `buyer_company_id, item_id, pricing_plan_id, status (trial/active/cancelled), start_date, end_date, seats`.
- `watchlist`: `buyer_user_id, item_id`.

## 5. Vendor domain

- `vendors`: `company_name, slug, description, logo_url, website_url, support_email, status (pending/active/suspended), verification_status (pending/verified/rejected)`
- `vendor_users`: `vendor_id, name, email, role (admin/sales/support)`
- `vendor_addresses`: `type (business/mailing)` plus postal fields.
- `vendor_verification_artifacts`: same shape as the buyer version, with `artifact_type (business_registration/tax_id/bank_verification/other)`.
- `payout_accounts`: `type (bank/other), provider_token, is_default`.
- `commission_rates`: `vendor_id, item_id (nullable), category_id (nullable), rate`.

## 6. Decisions and the reasoning behind them

| Decision | Chosen | Why |
|----------|--------|-----|
| Features as rows or a JSON blob | Structured table | "Comparison shopping" is a named use case, and you can't diff paragraphs |
| Item in one category or many | Many-to-many | A tool can be both CRM and Analytics |
| Pricing plans now or with billing | Separate table now | Plans are the sellable unit (SKU) and are shown on listing pages |
| Watchlist owner | Individual `buyer_users` | People browse on their own before involving colleagues |
| Subscriptions now or later | Now | Verified reviews and dashboards depend on knowing what was bought |
| Review references | Both company and user | Company drives `is_verified_purchase`; user is the author |
| Buyer verification | Optional, triggered (enterprise deals, RFPs, high spend) | Self-serve buyers should still be able to browse and buy lower tiers |
| Verification reviewer | Nullable `reviewed_by_staff_id`, no FK | The internal staff entity is deferred |
| Vendor addresses | Separate `vendor_addresses` table | Chosen over a polymorphic `addresses` table, to keep FKs simple and typed |
| Payouts | `payout_accounts` now, `payouts` deferred to billing | Payout records depend on revenue and commission data that doesn't exist yet |
| Commission rate | Separate `commission_rates` table | Allows per-item and per-category overrides from day one. The platform default lives in application config |
| Vendor `status` vs `verification_status` | Two fields | They move independently. A verified vendor can be suspended, and verification can be re-checked while a vendor stays active |
| Catalog filtering | Query-time join, no `is_orderable` flag on items | Suspending a vendor hides its listings instantly, with no bulk update and no race window |

Vendor KYB is a hard gate: a vendor can only be `active` while `verified`. Pending vendors can still draft listings, so they're ready to publish the moment they're approved, but those listings can never reach `published`.

## 7. Next steps

- Add the Part 2 auth and membership schema, which is designed but not migrated: `auth_provider_id` on both user tables, `sso_connection_id` on both company tables, and the `buyer_join_requests` / `vendor_join_requests` tables. See `isaas-platform/architecture/auth-and-membership-2026-09-27-20-55.md`.
- Still deferred: `promotions` (featured listings), `payouts`, an internal staff table, a `feature_key` normalization table, and "industries served" / "company sizes supported" on items.
- Use the real schema names (`buyer_companies.*`) in backend code, not the names from the plan.
