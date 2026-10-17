# Authentication, authorization and company membership

Scope: buyer companies and vendors as organizations with many members
Status: designed (Part 2 plan). **Not yet in the database migrations** as of `isaas-database` `v1.0.0`.

## 1. Principle

Auth0 says **who** someone is. Our schema and the Spring Boot backends decide **which company** they belong to and **what role** they have. See `isaas-platform/language-and-framework/auth-provider-2026-09-27-20-49.md` for the provider choice.

## 2. Three ways a person joins a company

1. **Bootstrap: the first person at a new company.** They create the company record and become its `admin`. No peer approval is possible, since nobody else is there yet. Trust comes from the company-level `verification_status` (KYC/KYB): a hard gate for vendors, optional and triggered for buyers.
2. **Peer approval: later employees, no SSO configured.** The person signs in through Auth0, finds their company and submits a join request. An existing `admin` at that company approves or rejects it. This is company-internal trust, so marketplace staff aren't involved, and it's kept separate from the KYC/KYB verification-artifact tables.
3. **SSO auto-provisioning: later employees, SSO configured.** If the company has an Auth0 enterprise connection (SAML/OIDC) with us, a person signing in through it is added immediately with the lowest role (`viewer`). Their identity provider vouches for their employment, but not for their permission level, so an admin upgrades them if needed.

Paths 2 and 3 are recorded in the same table, so there's one audit trail of how every member got access.

## 3. Schema additions (to be migrated)

On `buyer_companies.buyer_users` and `vendors.vendor_users`:

- `auth_provider_id`: the Auth0 subject (`sub`), used to map a JWT to a person.

On `buyer_companies.buyer_companies` and `vendors.vendors`:

- `sso_connection_id` (nullable): the Auth0 enterprise connection ID. NULL means no SSO, so peer approval is the only path for new members.

**`buyer_companies.buyer_join_requests`** (the plan wrote this as `buyers.` before the schema was named `buyer_companies`):
`id, buyer_company_id (FK), auth_provider_id, name, email, requested_role, status (pending/approved/rejected), granted_via (peer_approval/sso_auto_provision), reviewed_by_user_id (FK -> buyer_users, nullable), reviewed_at, review_notes, created_at`

**`vendors.vendor_join_requests`**:
`id, vendor_id (FK), auth_provider_id, name, email, requested_role, status (pending/approved/rejected), granted_via (peer_approval/sso_auto_provision), reviewed_by_user_id (FK -> vendor_users, nullable), reviewed_at, review_notes, created_at`

## 4. Rules

- `requested_role` is only the ask. The approving admin, or the SSO default of `viewer`, decides the role written to the user row, which is the only authoritative role.
- A user row is created only on approval, from the request's data, and `reviewed_by_user_id` records who vouched for the person.
- For SSO auto-provisioning: `status = approved` immediately, `granted_via = sso_auto_provision`, `reviewed_by_user_id = NULL`.
- No domain verification and no mandatory SSO at signup, because many vendors are solo founders without either.

## 5. Request flow in the backend

1. Validate the Auth0 JWT and read `sub` and the connection it came through.
2. If a `buyer_users` or `vendor_users` row has `auth_provider_id = sub`, load the company and role and authorize the request.
3. Otherwise, if the connection matches a company's `sso_connection_id`, auto-provision as `viewer` and write an approved join request.
4. Otherwise the person is unaffiliated. They can create a new company (bootstrap) or request to join one (peer approval).

## 6. Out of scope for now

- The SSO setup workflow itself (exchanging SAML/OIDC metadata with a company's identity provider through Auth0). Only the schema hooks are planned.
- Deriving roles from identity provider group claims.
- A long-term mechanism for Python-to-Spring-Boot service auth beyond a shared key.
- API contracts for join requests and approvals.

## 7. Next steps

- Write the migration in `isaas-database` with a `release:minor` label. It's additive: new nullable columns and two new tables. It also needs updates to `tests/11_enum_vocabulary.sql` for the new enums and a test per new rule.
- Decide whether `auth_provider_id` is unique and nullable (users seeded or invited before their first login won't have one yet).
- Then build sign-in and the join-request endpoints in the buyer and vendor backends.
