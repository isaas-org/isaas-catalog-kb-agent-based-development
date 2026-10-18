# Auth membership schema, verification gates and access tokens

Repo: `isaas-org/isaas-database`
Branch: `feature/FD-8_Auth_membership_schema`, PR #3 (label `release:minor`)
Design: `isaas-platform/architecture/authn-authz-design-2026-10-18-20-20.md` and `agent-auth-and-test-users-2026-10-18-20-20.md`

## 1. What was done

Six migrations on top of `v1.0.0`, one concern each:

```
20261018200000_create_platform_staff.sql                      platform schema, staff_users
20261018200100_add_identity_and_verification_columns.sql      auth_provider_id, user status, SSO, verified_at
20261018200200_add_pending_status_and_authority_artifacts.sql new enum values only
20261018200300_gate_buyer_company_activation.sql              buyer company, subscription, payment gates
20261018200400_create_join_requests.sql
20261018200500_create_access_tokens.sql
```

Plus five test files (`12` to `16`), four seed files (`05` to `08`), `scripts/bootstrap_staff.sh` and README updates.

## 2. Identity, staff and verification

`auth_provider_id` (the Auth0 `sub`) is nullable and unique on `vendor_users`, `buyer_users` and `platform.staff_users`. It stays NULL until first login, when the backend binds it by verified email. Users and staff get an `active`/`disabled` status so they are disabled rather than deleted.

Both company tables get `sso_connection_id`, `verified_by_staff_id` and `verified_at`. `verified_at` is stamped and cleared by a shared trigger, following how `items.enforce_publish_gate` stamps `published_at`:

```sql
IF NEW.verification_status::text = 'verified' THEN
    IF TG_OP = 'INSERT' OR OLD.verification_status IS DISTINCT FROM NEW.verification_status THEN
        NEW.verified_at := coalesce(NEW.verified_at, now());
    END IF;
ELSE
    NEW.verified_at := NULL;
    NEW.verified_by_staff_id := NULL;
END IF;
```

CHECKs keep the fields consistent when set directly (`*_verified_has_timestamp`, `*_verifier_requires_verified`), and `*_sso_requires_verified` allows an SSO connection only on a verified company. Existing verified vendors were backfilled with `verified_at = updated_at` and no verifier, since no staff member reviewed them.

Staff are referenced by FK from both artifact tables and both company tables, without cascade: staff with reviews can't be deleted.

## 3. Enum changes and why they have their own migration

| Enum | Change |
|------|--------|
| `buyer_companies.company_status` | `pending` added before `active` (now `pending, active, suspended`, same as vendors) |
| `vendors.artifact_type`, `buyer_companies.artifact_type` | `proof_of_authority`, `identity_document` added before `other` |
| New | `platform.staff_role`, `platform.staff_status`, `*_user_status`, `*.join_request_status`, `*.join_grant_path` (28 domain enums in total, all pinned in `11_enum_vocabulary.sql`) |

Postgres can't use an enum value in the same transaction that added it, and dbmate runs each migration in one transaction, so the values are added in `...200200` and used in `...200300`. Postgres also can't drop an enum value, so the down section casts the column to `text`, drops and recreates the type, and casts back. That down fails, by design, if any row still uses a new value.

## 4. Gates enforced by the database

| Rule | How |
|------|-----|
| Buyer company starts `pending`; `active` only while `verified` | default + CHECK `buyer_companies_active_requires_verified` |
| Trials and paid subscriptions need an active company (on insert, revival from `cancelled`, or move to another company) | trigger `buyer_companies.enforce_subscription_gate` |
| Payment methods need an active company | trigger `buyer_companies.enforce_payment_method_gate` |
| Join requests and SSO provisioning only reach an active company; admin invites also work while pending | triggers `vendors.enforce_join_request_gate`, `buyer_companies.enforce_join_request_gate` |

The subscription gate follows the publish gate: it locks the company row `FOR SHARE` and raises `check_violation`, so the backend can surface the message as-is.

```sql
IF NEW.status IN ('trial', 'active')
   AND (TG_OP = 'INSERT'
        OR OLD.status = 'cancelled'
        OR OLD.buyer_company_id IS DISTINCT FROM NEW.buyer_company_id) THEN
```

Cancelled subscriptions are history and pass. Active-but-unverified buyer companies are moved back to `pending` by the migration; Neon had none.

## 5. Join requests and access tokens

`vendor_join_requests` and `buyer_join_requests` record every member after the first. The path decides which fields must be set:

```sql
CONSTRAINT buyer_join_requests_path_rules CHECK (
    CASE granted_via
        WHEN 'sso_auto_provision' THEN status = 'approved' AND reviewed_by_user_id IS NULL
        WHEN 'admin_invite'       THEN status = 'approved' AND reviewed_by_user_id IS NOT NULL
        ELSE (status = 'pending') = (reviewed_by_user_id IS NULL)
    END)
```

Other rules: `auth_provider_id` may be NULL only for an invite; the reviewer belongs to the company (composite FK); one pending request per person per company (partial unique index). The reviewer FK has no `ON DELETE SET NULL`, because that would break the path CHECK; users are disabled, not deleted.

Access tokens (`vendor_access_tokens`, `buyer_access_tokens`) hold `token_prefix`, a 32-byte `token_hash` (CHECK `octet_length = 32`, UNIQUE), `expires_at` at most 90 days after creation, `last_used_at` and `revoked_at`. The owner is a composite FK to the users table with `ON DELETE CASCADE`.

## 6. Seeds, bootstrap and testing

- `05_buyer_companies.sql`: Harbor Health and Brightline Retail (active), Fernway Logistics (pending), each with an admin. Matched by name, since buyer companies have no natural unique key.
- `06_test_users.sql`: one member per role, plus Dana in a vendor and a buyer company under one email. Test data only.
- `07_verification_requests.sql`: pending registration, authority and ID documents for Cascade Labs and Fernway.
- `08_staff.sql`: the repo owner as staff admin, and as `verified_by_staff_id` on every seeded verified company. Added on the owner's review request; it sits in the seeds rather than a migration so migrations stay schema-only and test databases hold no personal data.
- `scripts/bootstrap_staff.sh`: `STAFF_EMAIL` / `STAFF_NAME` in, staff admin row out; re-runnable. For other staff, until the backend's staff endpoints exist.

Verified locally against `postgres:18` with dbmate v2.36.0: `test.sh` (16 files, full rollback, round trip) and `test_deploy.sh` pass. Every one of the 42 new constraints, indexes and triggers was dropped in a scratch database and its test failed. Upgrading a `v1.0.0` database that holds the old seed data works, and seeding twice leaves the same rows.

The test fixtures changed: `company.one` and `company.two` are now inserted `active`/`verified` (the new gates would block their subscriptions otherwise), and `company.pending`, `buyer_user.pending_admin` and `staff.reviewer` were added.

## 7. Next steps

- Repo owner merges PR #3 and approves the deploy (expected `v1.1.0`), then runs the seed workflow against Neon (which also adds the owner as staff).
- Backends (FD-7) map the new `check_violation` messages to API errors, as with the publish gate.
- Decide the lowest vendor role for SSO-provisioned members; the database doesn't constrain it yet.
